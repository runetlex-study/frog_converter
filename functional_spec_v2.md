# Функциональное задание v2: DOCX Summarizer
> Дополнение к [functional_spec.md](./functional_spec.md) — фиксирует все изменения, договорённости и исправления ошибок в процессе разработки.

---

## 1. Изменения по сравнению с v1

### 1.1 Замена API: DeepSeek → Qwen3-Omni-Flash

В исходном спеке был указан DeepSeek API. По решению заказчика используется **Qwen3-Omni-Flash** от Alibaba Cloud ModelStudio.

| Параметр | v1 (DeepSeek) | v2 (Qwen) |
|---|---|---|
| Endpoint | DeepSeek API | `https://dashscope-intl.aliyuncs.com/compatible-mode/v1` |
| Модель | `deepseek-chat` | `qwen3-omni-flash` |
| Переменная окружения | `DEEPSEEK_API_KEY` | `QWEN_API_KEY` |
| SDK | `httpx` (напрямую) | `httpx` (напрямую, без openai SDK) |

> **Важное ограничение Qwen3-Omni-Flash:** API поддерживает **только потоковый вывод** (`stream=True`). Ответ читается чанками из SSE-потока и склеивается в строку.

### 1.2 Замена openai SDK на прямые httpx-запросы

Изначально планировался `openai>=1.52.0`. В процессе тестирования обнаружено, что установленная версия `openai==2.24.0` вызывает `UnicodeEncodeError: 'ascii' codec` при кириллическом тексте в заголовках HTTP-запроса.

**Решение:** полностью убрали openai SDK, заменили на прямой вызов через `httpx.AsyncClient` с явным `json.dumps(payload, ensure_ascii=False).encode("utf-8")`.

### 1.3 Добавление фирменного персонажа (лягушка)

По запросу заказчика добавлен логотип/маскот — изображение `image_slide_ready_transparent.png` (лягушка в деловом костюме) в шапке интерфейса рядом с заголовком. Анимирован: hover-эффект + «wiggle» при нажатии кнопок.

Файл скопирован как `static/frog.png`.

### 1.4 Добавлен .env для хранения API-ключа

Для удобства локального тестирования добавлен `.env`-файл (загружается через `python-dotenv`).  
На Amvera ключ задаётся через веб-интерфейс (без `.env`).

### 1.5 Добавлен .gitignore

`.env` включён в `.gitignore` — ключ не попадает в репозиторий.

---

## 2. Уточнения по поведению `/summarize`

В исходном спеке `/summarize` принимал TXT-файлы. **Фактическая реализация:** принимает оригинальные **DOCX-файлы**, конвертирует их в текст прямо на сервере через `python-docx` — так же, как и `/convert`. Это сделано потому что фронтенд хранит оригинальные File-объекты и отправляет их напрямую.

Поток данных для суммаризации:
```
DOCX (браузер) → POST /summarize → python-docx → текст → Qwen API → CSV
```

---

## 3. Актуальная файловая структура

```
frog_converter/
│
├── main.py                              # FastAPI, три эндпоинта
├── requirements.txt                     # Зависимости Python
├── amvera.yml                           # Конфиг деплоя Amvera
├── .env                                 # API-ключ (локально, не в git)
├── .gitignore                           # Исключает .env и __pycache__
├── functional_spec.md                   # Исходное ТЗ
├── functional_spec_v2.md                # Этот файл
├── image_slide_ready_transparent.png   # Исходник логотипа
│
└── static/
    ├── index.html                       # Весь фронтенд
    └── frog.png                         # Логотип в интерфейсе
```

## 4. Актуальный requirements.txt

```
fastapi
uvicorn
python-docx
httpx
python-multipart
python-dotenv
```

> `openai` **не используется** (убран из-за несовместимости с кириллицей в версии 2.24.0).

---

## 5. Команда локального запуска

```bash
cd /path/to/frog_converter
# 1. Прописать ключ в .env: QWEN_API_KEY=sk-...
# 2. Установить зависимости (один раз):
pip3 install -r requirements.txt
# 3. Запустить сервер:
PYTHONUTF8=1 python3 -m uvicorn main:app --reload --port 8000
```

> `PYTHONUTF8=1` — обязательный флаг для корректной работы с кириллицей на Python 3.9 macOS.

---

## 6. Переменные окружения

| Переменная | Где задаётся | Значение |
|---|---|---|
| `QWEN_API_KEY` | `.env` (локально) | Ключ из [ModelStudio Console](https://modelstudio.console.alibabacloud.com/) → API Keys |
| `QWEN_API_KEY` | Amvera → «Переменные окружения» | То же значение |

---

## 7. Баги, выявленные и исправленные в процессе разработки

| # | Симптом | Причина | Исправление |
|---|---|---|---|
| 1 | Блок «Файлы не выбраны» съезжал влево | `<input type="file">` находился **внутри** `<label>` с flex-контейнером и нарушал выравнивание | Вынесли `<input>` **за пределы** `<label>`, CSS переделан с `display: block` |
| 2 | `'utf-8' codec can't decode bytes` в CSV | `/summarize` пытался декодировать загруженный DOCX-файл как UTF-8 текст | Добавили конвертацию через `python-docx` (как в `/convert`) |
| 3 | `'ascii' codec can't encode characters in position 7-14` | Заголовок HTTP-запроса `Authorization: Bearer <кириллический_placeholder>` — в `.env` не был вставлен реальный ключ | Вставили реальный ASCII-ключ из ModelStudio |
| 4 | Та же ошибка после вставки ключа | openai SDK 2.24.0 внутренне вызывает `.encode("ascii")` для заголовков при кириллическом контенте | Заменили openai SDK на прямые `httpx`-запросы с явной UTF-8 сериализацией |
