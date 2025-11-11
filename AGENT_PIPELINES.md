# Документация пайплайнов AI-агентов проекта JARVIS

Этот документ описывает все пайплайны обработки запросов в проекте JARVIS, включая последовательность вызовов промтов, передачу данных между этапами и схемы работы агентов.

---

## 1. HuggingGPT Pipeline - Мультимодальная оркестрация задач

HuggingGPT использует четырехэтапный пайплайн для обработки сложных мультимодальных запросов пользователя.

### 1.1 Общая архитектура

```
┌─────────────┐
│ User Input  │
└──────┬──────┘
       │
       ▼
┌─────────────────────────────────┐
│   STAGE 1: Task Parsing         │
│   (parse_task)                  │
│   Промт: parse_task_tprompt     │
└──────┬──────────────────────────┘
       │ Результат: JSON массив задач
       │ [{"task": "...", "id": 0, "dep": [-1], "args": {...}}]
       ▼
┌─────────────────────────────────┐
│   Preprocessing                 │
│   - unfold(tasks)               │
│   - fix_dep(tasks)              │
└──────┬──────────────────────────┘
       │ Обработанный список задач
       ▼
┌─────────────────────────────────┐
│   STAGE 2: Model Selection      │
│   (choose_model)                │
│   Промт: choose_model_tprompt   │
│   Для КАЖДОЙ задачи             │
└──────┬──────────────────────────┘
       │ Для каждой задачи: {"id": "model_id", "reason": "..."}
       ▼
┌─────────────────────────────────┐
│   STAGE 3: Task Execution       │
│   (model_inference)             │
│   Параллельно с зависимостями   │
└──────┬──────────────────────────┘
       │ Результаты выполнения всех задач
       │ {task_id: {"result": ...}, ...}
       ▼
┌─────────────────────────────────┐
│   STAGE 4: Response Generation  │
│   (response_results)            │
│   Промт: response_results_tprompt│
└──────┬──────────────────────────┘
       │
       ▼
┌─────────────┐
│   Response  │
│   to User   │
└─────────────┘
```

### 1.2 Stage 1: Task Parsing (Разбор задач)

**Функция:** `parse_task(context, input, api_key, api_type, api_endpoint)`

**Расположение:** `hugginggpt/server/awesome_chat.py:317`

**Входные данные:**
- `context`: История предыдущих сообщений (список)
- `input`: Текущий пользовательский запрос (строка)
- `api_key`, `api_type`, `api_endpoint`: Параметры API

**Промт структура:**
```python
messages = [
    {"role": "system", "content": parse_task_tprompt},
    # Few-shot примеры из parse_task_demos_or_presteps
    {"role": "user", "content": "Give you some pictures..."},
    {"role": "assistant", "content": "[{\"task\": \"image-to-text\"...}]"},
    # ...
    {"role": "user", "content": parse_task_prompt}  # Финальный запрос
]
```

**Параметры LLM:**
- `model`: text-davinci-003 / gpt-4
- `temperature`: 0 (детерминированный вывод)
- `logit_bias`: Повышение вероятности JSON токенов

**Выходные данные (формат JSON):**
```json
[
  {
    "task": "image-to-text",
    "id": 0,
    "dep": [-1],
    "args": {"image": "/path/to/image.jpg"}
  },
  {
    "task": "text-to-speech",
    "id": 1,
    "dep": [0],
    "args": {"text": "<GENERATED>-0"}
  }
]
```

**Обработка результата:**
1. Парсинг JSON строки
2. Если парсинг не удался → fallback на `chitchat()`
3. Если массив пустой `[]` → fallback на `chitchat()`
4. Если одна задача типа conversational → fallback на `chitchat()`
5. Иначе → переход к preprocessing

### 1.3 Preprocessing: Unfold & Fix Dependencies

**Функция:** `unfold(tasks)` и `fix_dep(tasks)`

**Назначение:** Разворачивание задач и исправление некорректных зависимостей

**Обработка:**
- `unfold()`: Разворачивает сложные задачи на подзадачи
- `fix_dep()`: Исправляет циклические и некорректные зависимости
- Проверка: `dep_id >= task["id"]` → устанавливается в `[-1]`

### 1.4 Stage 2: Model Selection (Выбор модели)

**Функция:** `choose_model(input, task, metas, api_key, api_type, api_endpoint)`

**Расположение:** `hugginggpt/server/awesome_chat.py:350`

**Вызывается:** Внутри `run_task()` для КАЖДОЙ задачи

**Входные данные:**
- `input`: Оригинальный пользовательский запрос
- `task`: Объект задачи `{"task": "image-to-text", "id": 0, ...}`
- `metas`: Список доступных моделей с описаниями

**Промт структура:**
```python
messages = [
    {"role": "system", "content": choose_model_tprompt},
    # Динамический few-shot пример
    {"role": "user", "content": "{{input}}"},
    {"role": "assistant", "content": "{{task}}"},
    # Финальный запрос
    {"role": "user", "content": choose_model_prompt}
]
```

**Параметры LLM:**
- `model`: text-davinci-003 / gpt-4
- `temperature`: 0
- `logit_bias`: Повышение вероятности для выбора модели (значение: 5)

**Выходные данные (формат JSON):**
```json
{
  "id": "nlpconnect/vit-gpt2-image-captioning",
  "reason": "This model is specifically designed for image captioning..."
}
```

**Процесс выбора модели:**
1. Получение кандидатов из `MODELS_MAP[task_type]`
2. Фильтрация по доступности (local/huggingface)
3. Топ-K моделей (default: 5) передаются в промт
4. LLM выбирает лучшую модель с обоснованием

### 1.5 Stage 3: Task Execution (Выполнение задач)

**Функция:** `run_task(input, task, results, api_key, api_type, api_endpoint)`

**Расположение:** `hugginggpt/server/awesome_chat.py` (в основном пайплайне)

**Параллельное выполнение с зависимостями:**

```python
while True:
    for task in tasks:
        dep = task["dep"]
        # Проверка: все зависимости выполнены?
        if dep[0] == -1 or all_deps_completed(dep, results):
            # Запуск в отдельном потоке
            thread = threading.Thread(target=run_task, args=(input, task, results, ...))
            thread.start()
            tasks.remove(task)

    if len(tasks) == 0:
        break  # Все задачи выполнены
```

**Схема выполнения задачи:**

```
┌────────────────────────┐
│  Задача с dep=[-1]     │
│  готова к выполнению   │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│  get_candidates()      │
│  Получить список       │
│  кандидатов моделей    │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│  get_avaliable_models()│
│  Проверить доступность │
│  (local/huggingface)   │
└───────────┬────────────┘
            │
            ▼
┌────────────────────────┐
│  choose_model()        │
│  [STAGE 2 PROMPT]      │
│  Выбрать лучшую модель │
└───────────┬────────────┘
            │ model_id
            ▼
┌────────────────────────┐
│  Подготовка аргументов │
│  Замена <GENERATED>-X  │
│  на результаты задач   │
└───────────┬────────────┘
            │ prepared_args
            ▼
┌────────────────────────┐
│  model_inference()     │
│  local / huggingface   │
└───────────┬────────────┘
            │ inference_result
            ▼
┌────────────────────────┐
│  Сохранить в results   │
│  results[task_id] = {  │
│    "task": {...},      │
│    "inference": {...}  │
│  }                     │
└────────────────────────┘
```

**Обработка зависимостей:**

Пример: `{"text": "<GENERATED>-0"}` означает использовать результат задачи с `id=0`

```python
# Поиск в результатах предыдущей задачи
resource = results[dep_task_id]["inference"]

# Извлечение нужного типа ресурса
if "generated text" in resource:
    args["text"] = resource["generated text"]
elif "generated image" in resource:
    args["image"] = resource["generated image"]
# и т.д.
```

### 1.6 Stage 4: Response Generation (Генерация ответа)

**Функция:** `response_results(input, results, api_key, api_type, api_endpoint)`

**Расположение:** `hugginggpt/server/awesome_chat.py:377`

**Входные данные:**
- `input`: Оригинальный пользовательский запрос
- `results`: Словарь всех результатов выполнения задач
  ```python
  {
    0: {"task": {...}, "inference": {"generated text": "..."}},
    1: {"task": {...}, "inference": {"generated image": "/images/x.png"}},
    ...
  }
  ```

**Промт структура:**
```python
messages = [
    {"role": "system", "content": response_results_tprompt},
    # Few-shot пример
    {"role": "user", "content": "{{input}}"},
    {"role": "assistant", "content": "Before give you a response, I want to introduce my workflow...{{processes}}..."},
    # Финальный запрос
    {"role": "user", "content": response_results_prompt}
]
```

**Параметры LLM:**
- `model`: text-davinci-003 / gpt-4
- `temperature`: 0

**Выходные данные:**
Естественноязыковой ответ пользователю, включающий:
- Прямой ответ на запрос
- Описание workflow (какие модели использовались)
- Пути к сгенерированным файлам (изображения, аудио, видео)
- Фильтрация нерелевантной информации

**Пример финального ответа:**
```
Sure. Based on your request, I have generated a canny image for you.

First, I used the image-to-text model (nlpconnect/vit-gpt2-image-captioning)
to convert /examples/f.jpg to text. The result is "a herd of giraffes
and zebras grazing in a field".

Then, I used the canny-text-to-image model (lllyasviel/sd-controlnet-canny)
to generate a canny image. The generated image is located at /images/a7b3.png.

Is there anything else I can help you with?
```

### 1.7 Полный пример: Обработка запроса с изображением

**Пользовательский запрос:**
```
"Look at /image.jpg, tell me what's in the picture and generate a similar image"
```

**Stage 1: Task Parsing Output**
```json
[
  {"task": "image-to-text", "id": 0, "dep": [-1], "args": {"image": "/image.jpg"}},
  {"task": "text-to-image", "id": 1, "dep": [0], "args": {"text": "<GENERATED>-0"}}
]
```

**Stage 2: Model Selection (для задачи 0)**

*Вход:*
- `task`: "image-to-text"
- `metas`: [nlpconnect/vit-gpt2-image-captioning, Salesforce/blip-image-captioning, ...]

*Выход:*
```json
{"id": "nlpconnect/vit-gpt2-image-captioning", "reason": "Best for general image captioning"}
```

**Stage 3: Task Execution**

*Задача 0 выполнена:*
```json
{
  "0": {
    "task": {"task": "image-to-text", ...},
    "inference": {"generated text": "a cat sitting on a couch"}
  }
}
```

*Задача 1 - подготовка аргументов:*
- Замена `<GENERATED>-0` → `"a cat sitting on a couch"`

**Stage 2: Model Selection (для задачи 1)**

*Вход:*
- `task`: "text-to-image"
- `metas`: [runwayml/stable-diffusion-v1-5, stabilityai/stable-diffusion-2, ...]

*Выход:*
```json
{"id": "runwayml/stable-diffusion-v1-5", "reason": "High quality text-to-image generation"}
```

**Stage 3: Task Execution (задача 1)**

*Задача 1 выполнена:*
```json
{
  "1": {
    "task": {"task": "text-to-image", ...},
    "inference": {"generated image": "/images/c2d4.png"}
  }
}
```

**Stage 4: Response Generation**

*Вход в промт:*
- `input`: "Look at /image.jpg, tell me what's in the picture and generate a similar image"
- `processes`: [результаты задач 0 и 1]

*Выход:*
```
I've analyzed your image. The picture shows a cat sitting on a couch.
Based on this description, I've generated a similar image for you using
the Stable Diffusion model. The new image is available at /images/c2d4.png.
```

### 1.8 Обработка ошибок и Edge Cases

**Chitchat Fallback:**
- Если task parsing вернул пустой массив
- Если JSON не распарсился
- Если единственная задача типа "conversational"

**Retry механизм:**
- Максимум 160 итераций ожидания (80 секунд)
- Если задача зависла → прерывание цикла

**Error handling:**
```python
if "error" in task_str:
    return {"message": task_str["error"]["message"]}
```

### 1.9 Конфигурация и параметры

**Ключевые параметры из config:**
- `num_candidate_models`: 5 (количество моделей для выбора)
- `max_description_length`: 100 (макс. длина описания модели)
- `inference_mode`: "hybrid" (local/huggingface/hybrid)
- `local_deployment`: "full" (minimal/standard/full)
- `logit_bias.parse_task`: 0.1
- `logit_bias.choose_model`: 5

**Поддерживаемые типы задач (34+):**
- NLP: text-classification, summarization, translation, question-answering, etc.
- CV: image-classification, object-detection, image-segmentation, text-to-image, etc.
- Audio: text-to-speech, speech-recognition, audio-classification
- Multimodal: visual-question-answering, document-question-answering
- ControlNet: canny-control, depth-control, openpose-control, etc.

---

