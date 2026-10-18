# Freedom Speech API: OpenAPI и Postman

Файлы в этой папке:

| Файл | Что это |
|---|---|
| `freedomspeech.yaml` | OpenAPI 3.1: все публичные HTTP-эндпоинты (распознавание, синтез, голоса, перевод, сервис) |
| `freedomspeech.postman_collection.json` | Postman-коллекция v2.1, по одному запросу на эндпоинт, переменные `baseUrl` и `apiKey` |
| этот README | как пользоваться и описание WebSocket-протоколов `/ws/asr` и `/ws/tts` (в OpenAPI их нет) |

Базовый адрес: `https://api.freedomspeech.kz`. Ключ выдаётся в личном кабинете.

Ключ из кабинета console.freedomspeech.kz работает на https://api.freedomspeech.kz, это адрес по умолчанию с 18.10.2026. На нём опубликованы `POST /v1/audio/transcriptions`, `POST /v1/audio/speech`, `WS /ws/asr` и `WS /ws/tts`. Остальные маршруты из `freedomspeech.yaml` (клонирование, voice design, перевод, каталог голосов, health) на этом адресе пока не опубликованы (404, бэкенд в работе) и ключам кабинета недоступны.

## Как пользоваться

Открыть спецификацию в Swagger UI, Redoc, Stoplight или Insomnia: все они понимают OpenAPI 3.1.
Для генерации клиента подходит `openapi-generator-cli` или `openapi-typescript`:

```bash
npx openapi-typescript freedomspeech.yaml -o freedomspeech.d.ts
```

Проверка спецификации (нужны пакеты `pyyaml` и `openapi-spec-validator`):

```bash
pip install pyyaml openapi-spec-validator
openapi-spec-validator freedomspeech.yaml
```

Postman: `File > Import`, выбрать `freedomspeech.postman_collection.json`, затем в переменных коллекции
заполнить `apiKey`. Файловые запросы (транскрипция, клон голоса) просят выбрать файл в теле запроса.

## Авторизация

Основной способ: заголовок `X-API-Key: <ключ>`. Он описан в `securitySchemes.apiKey` и стоит по умолчанию на всех защищённых путях.

Второй вариант, `Authorization: Bearer <ключ>`, описан как `securitySchemes.bearer`. Он принимается не на всех установках,
публичная документация описывает только `X-API-Key`.

Без ключа работают `GET /api/voices`, `GET /v2/languages`, `GET /api/asr/health`.

Запросы должны нести обычный браузерный `User-Agent`. Пустой или «серверный» User-Agent может получить 403 от защитного прокси.

## Коды ошибок

| Код | Смысл |
|---|---|
| 400 | неверный запрос (нет файла, пустой текст, неизвестный голос) |
| 401 | ключ не передан или не принят |
| 402 | закончился баланс или лимит тарифа |
| 404 | не найдено (например, голос для превью) |
| 413 | файл слишком большой |
| 429 | слишком много запросов, подождать и повторить |
| 500 | внутренняя ошибка сервиса |
| 502, 503 | речевой сервис недоступен или перегружен |

Тело ошибки: `{"detail": "..."}` или `{"error": "..."}`.

## WebSocket: потоковое распознавание `/ws/asr`

Адрес: `wss://api.freedomspeech.kz/ws/asr?apiKey=<ключ>&prettify=false`

Ключ передаётся в query-параметре `apiKey`. `prettify=true` включает расстановку пунктуации в итоговом тексте.

Порядок работы:

1. Клиент открывает соединение и ждёт кадр `ready`. До него аудио слать нельзя.
2. Клиент шлёт бинарные кадры: PCM16, 16 000 Гц, моно, little-endian, без WAV-заголовка. Удобный размер кадра: 100 мс (3200 байт).
3. Сервер по ходу присылает `partial` (промежуточный текст) и `final` (законченная фраза с языком).
4. Чтобы закрыть текущую фразу принудительно, клиент шлёт текстовый кадр `{"type": "stop"}`. Сокет остаётся открытым, можно продолжать.
5. Если 15 секунд не приходит аудио, сервер закрывает соединение.

Кадры сервер -> клиент:

```json
{"type": "ready"}
{"type": "partial", "text": "сәлем қалай"}
{"type": "final", "text": "Сәлем, қалайсың?", "language": "kk"}
{"type": "error", "message": "invalid api key"}
```

Кадры клиент -> сервер:

```
<binary> 3200 байт PCM16 16 кГц моно (100 мс)
<binary> ...
{"type": "stop"}
```

Ошибка авторизации может прийти либо как кадр `error`, либо как отказ при открытии соединения (HTTP 401 на апгрейде).

Пример на JavaScript (браузер):

```js
const ws = new WebSocket(`wss://api.freedomspeech.kz/ws/asr?apiKey=${key}&prettify=true`);
ws.binaryType = "arraybuffer";
ws.onmessage = (ev) => {
  const msg = JSON.parse(ev.data);
  if (msg.type === "ready") startMicrophone();          // дальше слать PCM16 16 кГц
  if (msg.type === "partial") showDraft(msg.text);
  if (msg.type === "final") appendLine(msg.text, msg.language);
  if (msg.type === "error") console.error(msg.message);
};
// из AudioWorklet: ws.send(int16Array.buffer)
```

## WebSocket: потоковый синтез `/ws/tts`

Адрес: `wss://api.freedomspeech.kz/ws/tts?apiKey=<ключ>`

Одно соединение обслуживает несколько запросов подряд. Каждый запрос помечается `requestId`, ответы несут тот же `requestId`.
Очередь на одном соединении: до 8 запросов; запрос ждёт своей очереди до 60 секунд, потом приходит `error` с `code: "queue_timeout"`.

Кадры клиент -> сервер:

```json
{"type": "synthesize", "text": "Сәлеметсіз бе!", "voice": "tomiris", "model": "freedomspeech-flash", "language": "kk", "requestId": "req-1"}
{"type": "cancel", "requestId": "req-1"}
{"type": "cancel"}
```

Поля `synthesize`: `text` (обязательно), `voice` (по умолчанию `tomiris`), `model` (`freedomspeech-flash` или `freedomspeech`),
`language` (`kk`, `ru`, `en`), `emotion` (пока не подключено для flash), `requestId` (любая строка; если не передать, сервер сопоставить ответы не сможет).
`cancel` без `requestId` отменяет все запросы соединения.

Кадры сервер -> клиент:

```json
{"type": "audio_chunk", "requestId": "req-1", "chunkIndex": 0, "audio": "<base64 PCM16>", "sampleRate": 24000, "format": "pcm16"}
{"type": "audio_chunk", "requestId": "req-1", "chunkIndex": 1, "audio": "<base64 PCM16>", "sampleRate": 24000, "format": "pcm16"}
{"type": "audio_end", "requestId": "req-1", "totalChunks": 2, "sampleRate": 24000}
{"type": "cancelled", "requestId": "req-1", "totalChunks": 1}
{"type": "error", "requestId": "req-1", "code": "queue_full", "message": "too many pending requests"}
{"type": "text_chunk", "requestId": "req-1", "text": "..."}
```

`audio` в `audio_chunk` это base64 от сырого PCM16 моно, частота в `sampleRate` (обычно 24 000 Гц). Чтобы получить WAV,
нужно склеить все куски и добавить 44-байтный заголовок. Кадры `text_chunk` служебные, их можно игнорировать.

Коды в `error`: `queue_full` (очередь из 8 занята), `queue_timeout` (ждали дольше 60 с), `upstream_failed` (сбой движка синтеза),
другие значения возможны и означают ошибку запроса.

Сколько ждать: первый `audio_chunk` для длинного текста может прийти не сразу, разумный таймаут ожидания кадра не меньше 90 секунд.

Пример на JavaScript (браузер, воспроизведение через Web Audio):

```js
const ws = new WebSocket(`wss://api.freedomspeech.kz/ws/tts?apiKey=${key}`);
const ctx = new AudioContext({ sampleRate: 24000 });
let t = ctx.currentTime;
ws.onopen = () => ws.send(JSON.stringify({ type: "synthesize", text: "Сәлеметсіз бе!", voice: "tomiris", requestId: "req-1" }));
ws.onmessage = (ev) => {
  const msg = JSON.parse(ev.data);
  if (msg.type !== "audio_chunk") return;
  const bytes = Uint8Array.from(atob(msg.audio), (c) => c.charCodeAt(0));
  const pcm = new Int16Array(bytes.buffer);
  const buf = ctx.createBuffer(1, pcm.length, msg.sampleRate);
  buf.getChannelData(0).set(Float32Array.from(pcm, (s) => s / 32768));
  const src = ctx.createBufferSource(); src.buffer = buf; src.connect(ctx.destination);
  src.start(t); t += buf.duration;
};
```

Готовые клиенты с этими протоколами: Python SDK (`../python`) и JavaScript SDK (`../js`).

---

# Freedom Speech API: OpenAPI and Postman (English)

Files in this folder:

| File | What it is |
|---|---|
| `freedomspeech.yaml` | OpenAPI 3.1 for every public HTTP endpoint (recognition, synthesis, voices, translation, service) |
| `freedomspeech.postman_collection.json` | Postman collection v2.1, one request per endpoint, variables `baseUrl` and `apiKey` |
| this README | usage notes and the WebSocket protocols `/ws/asr` and `/ws/tts` (not part of OpenAPI) |

Base URL: `https://api.freedomspeech.kz`. API keys are issued in the account dashboard.

Keys from the console.freedomspeech.kz dashboard work on https://api.freedomspeech.kz, the default since 2026-10-18. Published there: `POST /v1/audio/transcriptions`, `POST /v1/audio/speech`, `WS /ws/asr`, `WS /ws/tts`. The other routes in `freedomspeech.yaml` (cloning, voice design, translation, voice catalog, health) are not published on that host yet (404, backend work in progress) and are not available to console keys.

## Usage

Open the spec in Swagger UI, Redoc, Stoplight or Insomnia. For client generation use `openapi-generator-cli` or `openapi-typescript`:

```bash
npx openapi-typescript freedomspeech.yaml -o freedomspeech.d.ts
```

Validate the spec (requires `pyyaml` and `openapi-spec-validator`):

```bash
pip install pyyaml openapi-spec-validator
openapi-spec-validator freedomspeech.yaml
```

Postman: `File > Import`, pick `freedomspeech.postman_collection.json`, then fill `apiKey` in the collection variables.
File requests (transcription, voice clone) ask you to pick a file in the request body.

## Authentication

Primary: header `X-API-Key: <key>` (`securitySchemes.apiKey`, applied by default to all protected paths).

Secondary: `Authorization: Bearer <key>` (`securitySchemes.bearer`). It is not accepted on every installation;
the public documentation only describes `X-API-Key`.

No key is needed for `GET /api/voices`, `GET /v2/languages`, `GET /api/asr/health`.

Send a regular browser-like `User-Agent`. An empty or bare server User-Agent may be rejected with 403 by the protective proxy.

## Error codes

| Code | Meaning |
|---|---|
| 400 | bad request (missing file, empty text, unknown voice) |
| 401 | key missing or rejected |
| 402 | balance or plan limit exhausted |
| 404 | not found (for example a voice for preview) |
| 413 | file too large |
| 429 | too many requests, wait and retry |
| 500 | internal service error |
| 502, 503 | speech service unavailable or overloaded |

Error body: `{"detail": "..."}` or `{"error": "..."}`.

## WebSocket: streaming recognition `/ws/asr`

URL: `wss://api.freedomspeech.kz/ws/asr?apiKey=<key>&prettify=false`

The key goes in the `apiKey` query parameter. `prettify=true` adds punctuation to final text.

Flow:

1. Open the socket and wait for the `ready` frame. Do not send audio before it.
2. Send binary frames: PCM16, 16 000 Hz, mono, little-endian, no WAV header. A convenient frame is 100 ms (3200 bytes).
3. The server sends `partial` (interim text) and `final` (a finished phrase with its language).
4. To force the current phrase to finish, send a text frame `{"type": "stop"}`. The socket stays open.
5. After 15 seconds without audio the server closes the connection.

Server -> client frames:

```json
{"type": "ready"}
{"type": "partial", "text": "сәлем қалай"}
{"type": "final", "text": "Сәлем, қалайсың?", "language": "kk"}
{"type": "error", "message": "invalid api key"}
```

Client -> server frames:

```
<binary> 3200 bytes of PCM16 16 kHz mono (100 ms)
<binary> ...
{"type": "stop"}
```

An authentication failure arrives either as an `error` frame or as a rejected upgrade (HTTP 401).

## WebSocket: streaming synthesis `/ws/tts`

URL: `wss://api.freedomspeech.kz/ws/tts?apiKey=<key>`

One connection serves many requests. Each request carries a `requestId`; every reply echoes it.
Queue per connection: up to 8 requests; a request waits up to 60 seconds, then an `error` with `code: "queue_timeout"` is sent.

Client -> server frames:

```json
{"type": "synthesize", "text": "Сәлеметсіз бе!", "voice": "tomiris", "model": "freedomspeech-flash", "language": "kk", "requestId": "req-1"}
{"type": "cancel", "requestId": "req-1"}
{"type": "cancel"}
```

`synthesize` fields: `text` (required), `voice` (default `tomiris`), `model` (`freedomspeech-flash` or `freedomspeech`),
`language` (`kk`, `ru`, `en`), `emotion` (not wired yet for flash), `requestId` (any string; without it replies cannot be matched to requests).
`cancel` without `requestId` cancels every request on the connection.

Server -> client frames:

```json
{"type": "audio_chunk", "requestId": "req-1", "chunkIndex": 0, "audio": "<base64 PCM16>", "sampleRate": 24000, "format": "pcm16"}
{"type": "audio_end", "requestId": "req-1", "totalChunks": 2, "sampleRate": 24000}
{"type": "cancelled", "requestId": "req-1", "totalChunks": 1}
{"type": "error", "requestId": "req-1", "code": "queue_full", "message": "too many pending requests"}
{"type": "text_chunk", "requestId": "req-1", "text": "..."}
```

`audio` is base64 of raw PCM16 mono at `sampleRate` (usually 24 000 Hz). Concatenate the chunks and prepend a 44-byte header to get a WAV.
`text_chunk` frames are informational and can be ignored.

`error` codes: `queue_full` (8 requests already pending), `queue_timeout` (waited over 60 s), `upstream_failed` (synthesis engine failure);
other values mean a request error.

Waiting: the first `audio_chunk` for a long text may take a while; use a per-frame wait of at least 90 seconds.

Ready-made clients implementing both protocols: Python SDK (`../python`) and JavaScript SDK (`../js`).
