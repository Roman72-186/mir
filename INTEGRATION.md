# Интеграция определения ближайшего пункта в существующий Telegram-бот

## Суть

При выборе доставки бот запрашивает адрес у пользователя, геокодирует его через Яндекс Геокодер, вычисляет расстояние до 4 статичных пунктов по формуле Haversine и направляет заказ в ближайший.

---

## Шаг 1. Получить API-ключ Яндекс Геокодера

1. Перейти на https://developer.tech.yandex.ru/
2. Создать проект (или использовать существующий)
3. Подключить **API Геокодера** (HTTP Геокодер)
4. Скопировать API-ключ
5. Добавить в `.env` бота:

```env
YANDEX_GEOCODER_API_KEY=ваш_ключ
CITY_NAME=Калининград
CITY_BBOX=20.35,54.65~20.70,54.82
```

> **CITY_BBOX** — ограничивающий прямоугольник, покрывающий Калининград + Гурьевск (lon1,lat1~lon2,lat2).
> Без него Яндекс может находить адреса в других городах.
> Координаты bbox можно взять из Яндекс Карт.

---

## Шаг 2. Добавить файлы в проект

Скопировать 3 файла в проект бота. Пути произвольные, ниже — рекомендуемые.

### 2.1. `data/stores.json` — 4 пункта выдачи

```json
[
  {
    "id": 1,
    "name": "Пункт №1 — Центр Калининграда",
    "address": "Ленинский проспект, 30, Калининград",
    "lat": 54.7104,
    "lon": 20.4942
  },
  {
    "id": 2,
    "name": "Пункт №2 — Сельма",
    "address": "ул. Согласия, 2, Калининград",
    "lat": 54.7370,
    "lon": 20.4600
  },
  {
    "id": 3,
    "name": "Пункт №3 — Московский район",
    "address": "Московский проспект, 100, Калининград",
    "lat": 54.6890,
    "lon": 20.5150
  },
  {
    "id": 4,
    "name": "Пункт №4 — Гурьевск",
    "address": "ул. Калининградское шоссе, 5, Гурьевск",
    "lat": 54.7700,
    "lon": 20.6010
  }
]
```

> **Координаты**: Открыть Яндекс Карты → правый клик по точке → «Что здесь?» → скопировать координаты. Формат: `lat` (широта, ~54.xx), `lon` (долгота, ~20.xx).

### 2.2. `utils/geocoder.js` — Яндекс Геокодер

```javascript
const YANDEX_GEOCODER_URL = 'https://geocode-maps.yandex.ru/1.x/';

async function geocode(address) {
  const apiKey = process.env.YANDEX_GEOCODER_API_KEY;
  if (!apiKey) throw new Error('YANDEX_GEOCODER_API_KEY не задан');

  const city = process.env.CITY_NAME || '';
  const query = city ? `${city}, ${address}` : address;

  const params = new URLSearchParams({
    apikey: apiKey,
    geocode: query,
    format: 'json',
    results: '1',
    lang: 'ru_RU',
  });

  if (process.env.CITY_BBOX) {
    params.append('bbox', process.env.CITY_BBOX);
    params.append('rspn', '1');
  }

  const res = await fetch(`${YANDEX_GEOCODER_URL}?${params}`);
  if (!res.ok) throw new Error(`Яндекс Геокодер: HTTP ${res.status}`);

  const data = await res.json();
  const member = data?.response?.GeoObjectCollection?.featureMember?.[0];
  if (!member) return null;

  const [lon, lat] = member.GeoObject.Point.pos.split(' ').map(Number);
  const formatted = member.GeoObject.metaDataProperty.GeocoderMetaData.text;

  return { lat, lon, formatted };
}

module.exports = { geocode };
```

### 2.3. `utils/nearest-store.js` — Haversine + поиск ближайшего

```javascript
const stores = require('../data/stores.json');

function haversine(lat1, lon1, lat2, lon2) {
  const toRad = (d) => (d * Math.PI) / 180;
  const dLat = toRad(lat2 - lat1);
  const dLon = toRad(lon2 - lon1);
  const a =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) * Math.sin(dLon / 2) ** 2;
  return 6371 * 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
}

function formatDistance(km) {
  return km < 1 ? `${Math.round(km * 1000)} м` : `${km.toFixed(1)} км`;
}

/**
 * Принимает координаты пользователя, возвращает ближайший пункт.
 * @returns {{ id, name, address, lat, lon, distance, distanceText }}
 */
function findNearestStore(lat, lon) {
  let nearest = null;
  let minDist = Infinity;

  for (const store of stores) {
    const dist = haversine(lat, lon, store.lat, store.lon);
    if (dist < minDist) {
      minDist = dist;
      nearest = { ...store, distance: dist, distanceText: formatDistance(dist) };
    }
  }

  return nearest;
}

/**
 * Все пункты отсортированные по расстоянию
 */
function findAllSorted(lat, lon) {
  return stores
    .map((s) => ({
      ...s,
      distance: haversine(lat, lon, s.lat, s.lon),
      distanceText: formatDistance(haversine(lat, lon, s.lat, s.lon)),
    }))
    .sort((a, b) => a.distance - b.distance);
}

module.exports = { findNearestStore, findAllSorted };
```

---

## Шаг 3. Встроить в flow заказа

В месте, где пользователь выбирает **«Доставка»**, нужно:

1. Перевести бота в состояние ожидания адреса
2. Получить адрес (текст или геолокация)
3. Определить ближайший пункт
4. Привязать к заказу

### Псевдокод интеграции

```javascript
const { geocode } = require('./utils/geocoder');
const { findNearestStore } = require('./utils/nearest-store');

// --- Когда пользователь нажал «Доставка» ---
// Установить состояние: ожидаем адрес
userState[chatId] = { step: 'waiting_address', orderId: currentOrderId };

bot.sendMessage(chatId,
  '📍 Отправьте адрес доставки текстом\nили поделитесь геолокацией:',
  {
    reply_markup: {
      keyboard: [[{ text: '📍 Отправить геолокацию', request_location: true }]],
      resize_keyboard: true,
      one_time_keyboard: true,
    }
  }
);

// --- Обработка текстового адреса ---
// (внутри обработчика сообщений, где проверяется состояние)
if (userState[chatId]?.step === 'waiting_address' && msg.text) {
  try {
    const geo = await geocode(msg.text);
    if (!geo) {
      bot.sendMessage(chatId, '❌ Адрес не найден. Попробуйте точнее.');
      return;
    }

    const store = findNearestStore(geo.lat, geo.lon);
    await assignStoreToOrder(userState[chatId].orderId, store, geo);

  } catch (err) {
    console.error('Geocode error:', err);
    bot.sendMessage(chatId, '⚠️ Ошибка поиска адреса, попробуйте ещё раз.');
  }
}

// --- Обработка геолокации ---
if (userState[chatId]?.step === 'waiting_address' && msg.location) {
  const { latitude, longitude } = msg.location;
  const store = findNearestStore(latitude, longitude);
  await assignStoreToOrder(userState[chatId].orderId, store, {
    lat: latitude, lon: longitude, formatted: 'Геолокация'
  });
}

// --- Привязка пункта к заказу ---
async function assignStoreToOrder(orderId, store, userGeo) {
  // 1. Сохранить в БД / объект заказа
  //    order.pickup_store_id = store.id;
  //    order.delivery_address = userGeo.formatted;
  //    order.delivery_lat = userGeo.lat;
  //    order.delivery_lon = userGeo.lon;

  // 2. Сообщить пользователю
  const navUrl = `https://yandex.ru/maps/?rtext=${userGeo.lat},${userGeo.lon}~${store.lat},${store.lon}&rtt=auto`;

  await bot.sendMessage(chatId, [
    `✅ Ваш заказ направлен в ближайший пункт:`,
    ``,
    `🏪 *${store.name}*`,
    `📍 ${store.address} (${store.distanceText} от вас)`,
    ``,
    `🗺 [Маршрут в Яндекс Картах](${navUrl})`,
  ].join('\n'), { parse_mode: 'Markdown', disable_web_page_preview: true });

  // 3. Обновить состояние → перейти к следующему шагу заказа
  userState[chatId].step = 'next_step';
}
```

---

## Шаг 4. Добавить .env-переменные в Docker

Если бот запускается через Docker Compose — добавить переменные:

```yaml
services:
  bot:
    environment:
      - YANDEX_GEOCODER_API_KEY=${YANDEX_GEOCODER_API_KEY}
      - CITY_NAME=Калининград
      - CITY_BBOX=20.35,54.65~20.70,54.82
```

Или если используется `env_file: .env` — просто дописать в `.env`.

---

## Структура файлов (что добавляется)

```
существующий-бот/
├── data/
│   └── stores.json           ← НОВЫЙ: 4 пункта с координатами
├── utils/
│   ├── geocoder.js           ← НОВЫЙ: Яндекс Геокодер
│   └── nearest-store.js      ← НОВЫЙ: Haversine + findNearestStore
├── .env                      ← ИЗМЕНИТЬ: добавить 3 переменные
└── ...остальной код бота
```

---

## Чеклист перед деплоем

- [ ] API-ключ Яндекс Геокодера получен и добавлен в `.env`
- [ ] Координаты 4 пунктов в `stores.json` — реальные (проверены на карте)
- [ ] CITY_BBOX соответствует городу
- [ ] В обработчике заказов добавлен шаг запроса адреса
- [ ] Результат `findNearestStore()` сохраняется в заказ
- [ ] Docker-контейнер пересобран: `docker compose build && docker compose up -d`

---

## Лимиты Яндекс Геокодера

| Тариф | Лимит | Цена |
|-------|-------|------|
| Бесплатный | 1 000 запросов/день | 0 ₽ |
| Платный | от 25 000/день | от 3 000 ₽/мес |

Для бота суши 1000 запросов/день — более чем достаточно.
