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

## 2. TaskBench Pipelines - Бенчмарк планирования задач

TaskBench предоставляет два типа пайплайнов в зависимости от типа зависимостей между задачами: Resource Dependency и Temporal Dependency.

### 2.1 Общая архитектура

TaskBench использует одноэтапный пайплайн для генерации плана задач с автоматическим восстановлением при ошибках формата.

```
┌─────────────────┐
│  User Request   │
│  + Tool List    │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────┐
│   Task Planning Prompt          │
│   (resource OR temporal)        │
│   + Few-shot examples           │
└────────┬────────────────────────┘
         │ LLM генерирует JSON
         ▼
┌─────────────────────────────────┐
│   JSON Parsing Attempt          │
└────────┬────────────────────────┘
         │
         ├──[Success]──► Output: task_steps + task_nodes (+ task_links)
         │
         └──[JSON Error]
                 │
                 ▼
         ┌─────────────────────────┐
         │  Reformatting Prompt    │
         │  (fix invalid JSON)     │
         └────────┬────────────────┘
                  │
                  ▼
         ┌─────────────────────────┐
         │  Second Parsing Attempt │
         └────────┬────────────────┘
                  │
                  ├──[Success]──► Output
                  │
                  └──[Fail]──► ContentFormatError
```

### 2.2 Pipeline Type 1: Resource Dependency

**Назначение:** Для задач, где выходные данные одного инструмента используются как входные данные другого.

**Используется для:**
- HuggingFace models dataset
- Multimedia tasks dataset

**Схема работы:**

```
User Request: "Generate an image from text and then apply style transfer"

                        ┌──────────────────┐
                        │  Tool List       │
                        │  • text-to-image │
                        │  • image-to-image│
                        │  • style-transfer│
                        └────────┬─────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  PROMPT CONSTRUCTION                                    │
│  ═══════════════════                                    │
│                                                         │
│  # TOOL LIST #:                                         │
│  {"tool": "text-to-image", "input-type": ["text"],     │
│   "output-type": ["image"]}                            │
│  {"tool": "style-transfer", "input-type": ["image",    │
│   "image"], "output-type": ["image"]}                  │
│  ...                                                    │
│                                                         │
│  # GOAL #:                                              │
│  Generate task steps and task nodes in JSON format:    │
│  {"task_steps": [...], "task_nodes": [...]}            │
│                                                         │
│  # REQUIREMENTS #:                                      │
│  1. Task name from TOOL LIST                           │
│  2. Steps aligned with nodes                           │
│  3. Dependencies via argument tags                     │
│  4. Arguments align with input-type                    │
│                                                         │
│  [Few-shot examples if provided]                       │
│                                                         │
│  # USER REQUEST #: {user_request}                      │
│  # RESULT #:                                            │
└─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  LLM OUTPUT (JSON)                                      │
└─────────────────────────────────────────────────────────┘
{
  "task_steps": [
    "Generate image from text using text-to-image",
    "Apply style transfer to <node-0> using style-transfer"
  ],
  "task_nodes": [
    {
      "task": "text-to-image",
      "arguments": ["a beautiful landscape"]
    },
    {
      "task": "style-transfer",
      "arguments": ["<node-0>", "van gogh style"]
    }
  ]
}
```

**Формат выходных данных:**

```json
{
  "task_steps": [
    "Step 1 description",
    "Step 2 description"
  ],
  "task_nodes": [
    {
      "task": "tool_name_from_list",
      "arguments": [
        "direct_value",
        "<node-0>",  // Reference to output of task 0
        "user_provided_filename.jpg"
      ]
    }
  ]
}
```

**Ключевые особенности:**

1. **Зависимости через тег `<node-j>`:**
   - `<node-0>` ссылается на выход задачи с индексом 0
   - `<node-2>` ссылается на выход задачи с индексом 2

2. **Type matching:**
   - Проверка соответствия `output-type` предыдущей задачи и `input-type` текущей
   - Пример: `text-to-image` (output: image) → `image-classification` (input: image)

3. **Arguments format:**
   - Простой список значений: `["arg1", "arg2", "<node-0>"]`
   - Без явного указания имен параметров

### 2.3 Pipeline Type 2: Temporal Dependency

**Назначение:** Для задач, где важна последовательность выполнения (временная зависимость).

**Используется для:**
- Daily Life APIs dataset (реальные REST API)

**Схема работы:**

```
User Request: "Book a flight and then reserve a hotel for the same dates"

                        ┌──────────────────┐
                        │  Tool List       │
                        │  • search_flight │
                        │  • book_flight   │
                        │  • search_hotel  │
                        │  • book_hotel    │
                        └────────┬─────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  PROMPT CONSTRUCTION                                    │
│  ═══════════════════                                    │
│                                                         │
│  # TOOL LIST #:                                         │
│  {"tool": "search_flight", "parameters": ["origin",    │
│   "destination", "date"]}                              │
│  {"tool": "book_flight", "parameters": ["flight_id"]}  │
│  ...                                                    │
│                                                         │
│  # GOAL #:                                              │
│  Generate task steps, task nodes AND task links:       │
│  {"task_steps": [...], "task_nodes": [...],            │
│   "task_links": [...]}                                 │
│                                                         │
│  # REQUIREMENTS #:                                      │
│  1. Task name from TOOL LIST                           │
│  2. Steps aligned with nodes                           │
│  3. task_links reflect temporal order                  │
│                                                         │
│  # USER REQUEST #: {user_request}                      │
│  # RESULT #:                                            │
└─────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────┐
│  LLM OUTPUT (JSON)                                      │
└─────────────────────────────────────────────────────────┘
{
  "task_steps": [
    "Step 1: Call search_flight with origin: 'NYC' and destination: 'LAX'",
    "Step 2: Call book_flight with flight_id: 'search_flight result'"
  ],
  "task_nodes": [
    {
      "task": "search_flight",
      "arguments": [
        {"name": "origin", "value": "NYC"},
        {"name": "destination", "value": "LAX"}
      ]
    },
    {
      "task": "book_flight",
      "arguments": [
        {"name": "flight_id", "value": "search_flight"}
      ]
    }
  ],
  "task_links": [
    {"source": "search_flight", "target": "book_flight"}
  ]
}
```

**Формат выходных данных:**

```json
{
  "task_steps": [
    "Step 1: Call tool_a with param1: 'value1' and param2: 'value2'",
    "Step 2: Call tool_b with param1: 'tool_a result'"
  ],
  "task_nodes": [
    {
      "task": "tool_a",
      "arguments": [
        {"name": "param1", "value": "value1"},
        {"name": "param2", "value": "value2"}
      ]
    },
    {
      "task": "tool_b",
      "arguments": [
        {"name": "param1", "value": "tool_a"}  // Reference by tool name
      ]
    }
  ],
  "task_links": [
    {"source": "tool_a", "target": "tool_b"}
  ]
}
```

**Ключевые особенности:**

1. **Task Links (временные связи):**
   ```json
   {"source": "task_i", "target": "task_j"}
   ```
   Означает: task_j должен выполниться ПОСЛЕ task_i

2. **Зависимости через имя инструмента:**
   - `"value": "search_flight"` означает использовать результат инструмента search_flight
   - Не через индекс, а через имя задачи

3. **Named Arguments:**
   - Каждый аргумент - объект с `name` и `value`
   - `[{"name": "param", "value": "val"}]`

4. **Explicit Step Format:**
   - Строгий формат: `"Step X: Call tool_name with param: 'value'"`

### 2.4 JSON Reformatting (Error Recovery)

Когда первая попытка парсинга JSON не удается, TaskBench автоматически пытается исправить формат.

**Схема восстановления:**

```
┌──────────────────────────┐
│  Invalid JSON detected   │
│  by json.loads()         │
└───────────┬──────────────┘
            │
            ▼
    ┌──────────────────┐
    │  reformat == True?│
    └───────┬──────────┘
            │
      [YES] │
            ▼
┌──────────────────────────────────────┐
│  Reformatting Prompt                 │
│  ════════════════════                │
│                                      │
│  "Please format # RESULT # to       │
│  strict JSON format                 │
│  # STRICT JSON FORMAT #"            │
│                                      │
│  Requirements:                       │
│  1. Don't change meaning            │
│  2. Ensure json.loads() compatible  │
│  3. Match required schema           │
│                                      │
│  # RESULT #: {illegal_result}       │
│  # STRICT JSON FORMAT #:            │
└───────────┬──────────────────────────┘
            │
            ▼
┌──────────────────────────────────────┐
│  Second LLM Call                     │
│  (using same or different model)    │
└───────────┬──────────────────────────┘
            │
            ▼
    ┌───────────────┐
    │ Parse attempt │
    └───────┬───────┘
            │
            ├──[Success]──► Return result
            │
            └──[Fail]──► Raise ContentFormatError
```

**Пример исправления:**

*Исходный невалидный JSON:*
```
{task_steps: ["Step 1", "Step 2"], task_nodes: [{task: "tool1", arguments: ["arg1"]}]}
```

*Промт для исправления:*
```
Please format the result # RESULT # to a strict JSON format.

Requirements:
1. Do not change the meaning
2. Ensure json.loads() compatible
3. Output must match schema

# RESULT #: {task_steps: [...], task_nodes: [...]}
# STRICT JSON FORMAT #:
```

*Исправленный JSON:*
```json
{"task_steps": ["Step 1", "Step 2"], "task_nodes": [{"task": "tool1", "arguments": ["arg1"]}]}
```

### 2.5 Конфигурация и параметры

**LLM параметры:**
- `model`: "gpt-4" / "gpt-3.5-turbo"
- `temperature`: 0.2 (небольшая случайность)
- `top_p`: 0.1 (фокус на наиболее вероятных токенах)
- `max_tokens`: 2000
- `frequency_penalty`: 0
- `presence_penalty`: 1.05 (избегать повторений)

**Few-shot примеры:**
- HuggingFace dataset: 3 примера (IDs: 10523150, 14611002, 22067492)
- Multimedia dataset: 3 примера (IDs: 30934207, 20566230, 19003517)
- Daily Life APIs: 3 примера (IDs: 38563456, 27267145, 91005535)

**Опции:**
- `use_demos`: 0-3 (количество few-shot примеров)
- `reformat`: true/false (включить автоисправление JSON)
- `reformat_by`: "self" / model_name (модель для исправления)

### 2.6 Полный пример: Resource Dependency

**Входные данные:**

```python
user_request = "Detect objects in image.jpg and describe what you see"
tool_list = [
    {"tool": "object-detection", "input-type": ["image"], "output-type": ["detection"]},
    {"tool": "image-to-text", "input-type": ["image"], "output-type": ["text"]},
    {"tool": "text-classification", "input-type": ["text"], "output-type": ["label"]}
]
```

**Промт (упрощенно):**

```
# TASK LIST #:
{"tool": "object-detection", "input-type": ["image"], "output-type": ["detection"]}
{"tool": "image-to-text", "input-type": ["image"], "output-type": ["text"]}

# GOAL #: Generate task steps and task nodes...
# USER REQUEST #: Detect objects in image.jpg and describe what you see
# RESULT #:
```

**Выход LLM:**

```json
{
  "task_steps": [
    "Detect objects in the image using object-detection",
    "Generate description of the image using image-to-text"
  ],
  "task_nodes": [
    {
      "task": "object-detection",
      "arguments": ["image.jpg"]
    },
    {
      "task": "image-to-text",
      "arguments": ["image.jpg"]
    }
  ]
}
```

### 2.7 Полный пример: Temporal Dependency

**Входные данные:**

```python
user_request = "Search for flights from NYC to SF and book the cheapest one"
tool_list = [
    {"tool": "search_flights", "parameters": ["origin", "destination"]},
    {"tool": "get_flight_price", "parameters": ["flight_id"]},
    {"tool": "book_flight", "parameters": ["flight_id"]}
]
```

**Промт (упрощенно):**

```
# TASK LIST #:
{"tool": "search_flights", "parameters": ["origin", "destination"]}
{"tool": "get_flight_price", "parameters": ["flight_id"]}
{"tool": "book_flight", "parameters": ["flight_id"]}

# GOAL #: Generate task steps, task nodes, and task links...
# USER REQUEST #: Search for flights from NYC to SF and book the cheapest one
# RESULT #:
```

**Выход LLM:**

```json
{
  "task_steps": [
    "Step 1: Call search_flights with origin: 'NYC' and destination: 'SF'",
    "Step 2: Call get_flight_price with flight_id: 'search_flights result'",
    "Step 3: Call book_flight with flight_id: 'cheapest flight from step 2'"
  ],
  "task_nodes": [
    {
      "task": "search_flights",
      "arguments": [
        {"name": "origin", "value": "NYC"},
        {"name": "destination", "value": "SF"}
      ]
    },
    {
      "task": "get_flight_price",
      "arguments": [
        {"name": "flight_id", "value": "search_flights"}
      ]
    },
    {
      "task": "book_flight",
      "arguments": [
        {"name": "flight_id", "value": "get_flight_price"}
      ]
    }
  ],
  "task_links": [
    {"source": "search_flights", "target": "get_flight_price"},
    {"source": "get_flight_price", "target": "book_flight"}
  ]
}
```

### 2.8 Сравнение двух типов пайплайнов

| Аспект | Resource Dependency | Temporal Dependency |
|--------|---------------------|---------------------|
| **Тип зависимости** | По типу данных (image→text) | По времени (A должно быть до B) |
| **Формат аргументов** | Простой список `["arg1", "<node-0>"]` | Named `[{"name": "x", "value": "y"}]` |
| **Ссылка на результат** | По индексу `<node-0>` | По имени инструмента `"tool_name"` |
| **Task Links** | Нет (неявные через args) | Да, явные `{"source": "A", "target": "B"}` |
| **Type checking** | Да (input-type/output-type) | Нет |
| **Применение** | NLP/CV конвейеры | REST API orchestration |
| **Step format** | Произвольный текст | Строгий: "Step X: Call..." |

### 2.9 Обработка ошибок

**Типы ошибок:**

1. **RateLimitError (429):** Слишком много запросов к API
2. **ContentFormatError:** JSON не валиден даже после reformatting
3. **JSONDecodeError:** Первичная ошибка парсинга (триггер reformatting)

**Error handling flow:**

```python
try:
    content = json.loads(response)
    return content
except JSONDecodeError:
    if reformat:
        # Попытка исправления
        reformatted = call_reformat_prompt(content)
        try:
            return json.loads(reformatted)
        except JSONDecodeError:
            raise ContentFormatError
    else:
        raise ContentFormatError
```

---

## 3. EasyTool Pipelines - API оркестрация и выполнение

EasyTool предоставляет четыре различных пайплайна для работы с API инструментами, каждый оптимизирован для своего типа задач.

### 3.1 Обзор пайплайнов

```
┌──────────────────────────────────────────────────────────────┐
│                    EasyTool Framework                        │
└──────────────────────────────────────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   FuncQA        │  │   ToolBench     │  │   RestBench     │
│   Multi-hop     │  │   API Selection │  │   Task Decomp   │
└─────────────────┘  └─────────────────┘  └─────────────────┘
        │                     │                     │
        │           ┌─────────┴─────────┐          │
        │           │                   │          │
        ▼           ▼                   ▼          ▼
    Task         Tool              Tool + API    Task
    Decompose    Selection         with RAG      Decompose
    + Topology   + API Choice      (Retrieval)   for REST
                 + Parameters
```

**Сравнение пайплайнов:**

| Пайплайн | Назначение | Декомпозиция | Зависимости | Retrieval |
|----------|-----------|--------------|-------------|-----------|
| **FuncQA-MH** | Математические функции (Multi-Hop) | ✅ Task Decompose + Topology | ✅ Через previous_log | ❌ |
| **FuncQA-OH** | Математические функции (One-Hop) | ❌ | ❌ | ❌ |
| **ToolBench** | API инструменты | ❌ | ✅ Sequential | ❌ |
| **ToolBench-RAG** | API инструменты с retrieval | ❌ | ✅ Sequential | ✅ Semantic search |
| **RestBench** | REST API (Spotify, TMDB) | ✅ Task Decompose | ❌ | ❌ |

---

### 3.2 Pipeline 1: FuncQA Multi-Hop (Мультихоп функции)

**Назначение:** Решение сложных вопросов, требующих последовательного вызова нескольких функций с зависимостями.

**Пример задачи:** "Convert 23 km/h to km/min and then multiply by 45"

**Полная схема пайплайна:**

```
┌──────────────────┐
│  User Question   │
│  + Tool List     │
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────┐
│  STAGE 1: Task Decomposition    │
│  Промт: task_decompose          │
│  Разбить вопрос на подзадачи    │
└────────┬────────────────────────┘
         │ Output: {"Tasks": ["Task 1", "Task 2", ...]}
         ▼
┌─────────────────────────────────┐
│  STAGE 2: Task Topology         │
│  Промт: task_topology           │
│  Определить зависимости         │
└────────┬────────────────────────┘
         │ Output: [{"task": "...", "id": 0, "dep": [-1]}, ...]
         ▼
         │
         ├──► Цикл по задачам (в порядке зависимостей)
         │
         └───────┐
                 ▼
         ┌───────────────────────────┐
         │  Задача без зависимостей  │
         │  или с выполненными dep   │
         └───────┬───────────────────┘
                 │
                 ▼
         ┌─────────────────────────────────┐
         │  STAGE 3A: Tool Selection       │
         │  Промт: choose_tool             │
         │  Выбрать инструмент             │
         └────────┬────────────────────────┘
                  │ Output: {"ID": tool_id}
                  ▼
         ┌─────────────────────────────────┐
         │  STAGE 3B: API Selection        │
         │  (если нужно)                   │
         └────────┬────────────────────────┘
                  │
                  ▼
         ┌─────────────────────────────────┐
         │  STAGE 3C: Parameter Extraction │
         │  Промт: choose_parameter        │
         │  или choose_parameter_depend    │
         └────────┬────────────────────────┘
                  │ Output: {"Parameters": {...}}
                  ▼
         ┌─────────────────────────────────┐
         │  STAGE 4: Function Execution    │
         │  Call_function()                │
         └────────┬────────────────────────┘
                  │ Result stored in previous_log
                  ▼
         ┌─────────────────────────────────┐
         │  Добавить результат в           │
         │  previous_log для след. задачи  │
         └────────┬────────────────────────┘
                  │
                  └───► Следующая задача (если есть)

         После выполнения всех задач:
                  │
                  ▼
         ┌─────────────────────────────────┐
         │  Финальный ответ из             │
         │  последней задачи               │
         └─────────────────────────────────┘
```

**Детальные этапы:**

#### Stage 1: Task Decomposition

**Промт:** `task_decompose(question, Tool_dic, model_name)`

**Входные данные:**
```python
question = "Convert 23 km/h to X km/min by 'divide_', then multiply X by 45 to get Y"
Tool_dic = [
    {"name": "divide_", "description": "Divide two numbers"},
    {"name": "multiply_", "description": "Multiply two numbers"}
]
```

**Промт структура:**
```
You need to decompose a complex user's question into some simple subtasks.

This is the user's question: {question}
This is tool list: {Tool_list}

Please note that:
1. Decompose into simple subtasks with one tool each
2. Write dependencies clearly (use X, Y variables)
3. Output in JSON format: {"Tasks": [...]}

Output:
```

**Выход LLM:**
```json
{
  "Tasks": [
    "Convert 23 km/h to X km/min by 'divide_'",
    "Multiply X km/min by 45 min to get Y by 'multiply_'"
  ]
}
```

#### Stage 2: Task Topology

**Промт:** `task_topology(question, task_ls, model_name)`

**Входные данные:**
- `question`: Оригинальный вопрос
- `task_ls`: Результат Stage 1

**Промт структура:**
```
Given a complex user's question, I have decomposed it into subtasks.
There exists logical connections and order among the tasks.

Output in JSON format:
[{"task": task, "id": task_id, "dep": [dependency_ids]}]

The "dep" field denotes ids of previous tasks that generate resources.
If no dependencies, set "dep" to -1.

This is user's question: {question}
These are subtasks: {task_ls}
Output:
```

**Выход LLM:**
```json
[
  {
    "task": "Convert 23 km/h to X km/min by 'divide_'",
    "id": 0,
    "dep": [-1]
  },
  {
    "task": "Multiply X km/min by 45 min to get Y by 'multiply_'",
    "id": 1,
    "dep": [0]
  }
]
```

#### Stage 3: Tool Selection & Parameter Extraction

**Для задачи 0 (без зависимостей):**

**3A. Tool Selection:**
```python
tool_id = choose_tool(task["task"], Tool_dic, tool_used, model_name)
# Output: {"ID": 1}  // divide_ tool
```

**3C. Parameter Extraction:**
```python
parameters = choose_parameter(API_instruction, api, api_dic, task["task"], model_name)
# Output: {"Parameters": {"input": [23, 60]}}  # 23 km/h ÷ 60 = km/min
```

**Для задачи 1 (с зависимостями):**

**3C. Parameter Extraction with Dependencies:**
```python
# previous_log содержит результат задачи 0
previous_log = "Task 0 result: 0.383 km/min"

parameters = choose_parameter_depend(
    API_instruction, api, api_dic,
    task["task"], model_name, previous_log
)
# Output: {"Parameters": {"input": [0.383, 45]}}  # используется результат задачи 0
```

#### Stage 4: Function Execution

```python
result = Call_function(function_name, parameters, task_id)
# Выполняет функцию и возвращает результат

# Для задачи 0: 23 / 60 = 0.383
# Для задачи 1: 0.383 * 45 = 17.235
```

**Формат previous_log:**
```python
previous_log = f"""
Task {task_id}:
Question: {task["task"]}
Function: {function_name}
Parameters: {parameters}
Result: {result}
"""
```

---

### 3.3 Pipeline 2: FuncQA One-Hop (Однохоп функции)

**Назначение:** Простые вопросы, требующие одного вызова функции.

**Упрощенная схема:**

```
┌──────────────────┐
│  User Question   │
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Tool Selection                 │
│  Промт: choose_tool             │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Parameter Extraction           │
│  Промт: choose_parameter        │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  Function Execution             │
└────────┬────────────────────────┘
         │
         ▼
┌──────────────────┐
│  Final Answer    │
└──────────────────┘
```

**Отличия от Multi-Hop:**
- ❌ Нет Stage 1 (Task Decomposition)
- ❌ Нет Stage 2 (Task Topology)
- ❌ Нет циклической обработки зависимостей
- ✅ Прямой путь: Tool → Parameters → Execute → Answer

---

### 3.4 Pipeline 3: ToolBench (API оркестрация)

**Назначение:** Последовательный вызов нескольких API для решения задачи.

**Полная схема:**

```
┌──────────────────┐
│  User Question   │
│  + Tool Catalog  │
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────┐
│  STAGE 1: Tool Selection        │
│  Промт: choose_tool             │
│  Выбрать ОДИН инструмент        │
└────────┬────────────────────────┘
         │ Output: {"ID": tool_id}
         ▼
┌─────────────────────────────────┐
│  Load Tool Instruction          │
│  tool_instruction.json          │
└────────┬────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  STAGE 2: API Selection         │
│  Промт: choose_API              │
│  Выбрать несколько API          │
│  из инструмента                 │
└────────┬────────────────────────┘
         │ Output: ["api1", "api2", ...]
         │
         └───► Цикл по выбранным API
                │
                ▼
         ┌─────────────────────────────────┐
         │  STAGE 3: Parameter Extraction  │
         │  Промт: choose_parameter        │
         │  (или _depend для последующих)  │
         └────────┬────────────────────────┘
                  │ Output: {"Parameters": {...}}
                  ▼
         ┌─────────────────────────────────┐
         │  STAGE 4: API Execution         │
         │  Call_function()                │
         └────────┬────────────────────────┘
                  │
                  ▼
         ┌─────────────────────────────────┐
         │  STAGE 5: Answer Generation     │
         │  Промт: answer_generation       │
         │  (или _depend)                  │
         └────────┬────────────────────────┘
                  │
                  ├──► Следующий API (если есть)
                  │
                  └──► Финальный ответ
```

**Ключевые особенности:**

1. **Последовательная обработка API:**
   - Первый API: используется `choose_parameter`
   - Последующие API: используется `choose_parameter_depend` с логами

2. **Answer Generation для каждого API:**
   ```python
   answer = answer_generation(question, API_instruction, call_result, model_name)
   # Преобразует технический результат в естественный язык
   ```

3. **Cumulative previous_log:**
   ```python
   previous_log += f"""
   API: {api_name}
   Parameters: {parameters}
   Result: {call_result}
   Answer: {answer}
   """
   ```

**Пример выполнения:**

*Вопрос:* "Search for hotels in Paris and get reviews for the top result"

*Stage 1 - Tool Selection:*
```json
{"ID": 15}  // "Hotels" tool
```

*Stage 2 - API Selection:*
```json
["search_hotels", "get_hotel_reviews"]
```

*Stage 3 - Parameters (API 1):*
```json
{"Parameters": {"location": "Paris", "limit": 10}}
```

*Stage 4 - Execution (API 1):*
```json
{"hotels": [{"id": "hotel_123", "name": "Le Hotel", ...}, ...]}
```

*Stage 5 - Answer Generation (API 1):*
```
"I found 10 hotels in Paris. The top result is Le Hotel (ID: hotel_123)."
```

*Stage 3 - Parameters (API 2) with dependency:*
```python
# previous_log содержит результат API 1
{"Parameters": {"hotel_id": "hotel_123"}}  # Извлечено из previous_log
```

*Stage 4 - Execution (API 2):*
```json
{"reviews": [{"rating": 4.5, "text": "Great location!"}, ...]}
```

*Stage 5 - Final Answer Generation:*
```
"The reviews for Le Hotel are mostly positive with an average rating of 4.5..."
```

---

### 3.5 Pipeline 4: ToolBench with RAG (Retrieval-Augmented)

**Назначение:** То же, что ToolBench, но с семантическим поиском релевантных API.

**Отличия от обычного ToolBench:**

```
┌──────────────────────────────────────────────────────┐
│  STAGE 1: Tool Selection (same as ToolBench)         │
└────────┬─────────────────────────────────────────────┘
         │
         ▼
┌──────────────────────────────────────────────────────┐
│  NEW: Semantic Retrieval                             │
│  ══════════════════════                              │
│  1. Embed user question                              │
│  2. Compute similarity with all APIs in tool         │
│  3. Select top-K most relevant APIs                  │
└────────┬─────────────────────────────────────────────┘
         │ Filtered API list (top-K relevant)
         ▼
┌──────────────────────────────────────────────────────┐
│  STAGE 2: API Selection (with filtered list)        │
│  Промт: choose_API                                   │
│  Выбрать из релевантных API                         │
└────────┬─────────────────────────────────────────────┘
         │
         └──► Rest is same as ToolBench
```

**Semantic Retrieval детали:**

```python
# 1. Получить эмбеддинги всех API
api_descriptions = [api["description"] for api in tool_apis]
api_embeddings = get_embeddings(api_descriptions)

# 2. Эмбеддинг вопроса
question_embedding = get_embedding(question)

# 3. Вычислить косинусное сходство
similarities = cosine_similarity([question_embedding], api_embeddings)[0]

# 4. Топ-K релевантных
top_k_indices = similarities.argsort()[-retrieval_num:][::-1]
filtered_apis = [tool_apis[i] for i in top_k_indices]

# 5. Передать только релевантные API в промт
```

**Преимущества:**
- Уменьшение контекста промта (только релевантные API)
- Более точный выбор API
- Работает с инструментами, имеющими 100+ API

---

### 3.6 Pipeline 5: RestBench (REST API декомпозиция)

**Назначение:** Работа с реальными REST API (Spotify, TMDB) через декомпозицию задач.

**Полная схема:**

```
┌──────────────────┐
│  User Question   │
│  + API List      │
└────────┬─────────┘
         │
         ▼
┌─────────────────────────────────┐
│  STAGE 1: Task Decomposition    │
│  Промт: task_decompose          │
│  С указанием ID инструментов    │
└────────┬────────────────────────┘
         │ Output: [{"Task": "...", "ID": 15}, ...]
         ▼
┌─────────────────────────────────┐
│  STAGE 2: Execution Loop        │
│  Для каждой подзадачи           │
└────────┬────────────────────────┘
         │
         └───► Сохранить путь инструментов

         ▼
┌──────────────────┐
│  Output:         │
│  tool_choice_ls  │
└──────────────────┘
```

**Промт для Task Decomposition:**

```
We have spotify database and the following tools:
{Tool_dic}

You need to decompose a complex user's question into simple subtasks
and let the model execute it step by step with these tools.

Please note that:
1. Break down into subtasks using tools mentioned above
2. List the ID of the tool used for each subtask
3. If no tool needed, use {"ID": -1}
4. Consider logical connections, order and constraints
5. Output in JSON format:
   [{"Task": "...", "ID": 15}, {"Task": "...", "ID": 19}]

Example:
Question: Pause the player
Output: [
  {"Task": "Get current playback state", "ID": 15},
  {"Task": "Pause playback", "ID": 19}
]

This is the user's question: {question}
Output:
```

**Выход:**

```json
[
  {"Task": "Search for song 'Bohemian Rhapsody'", "ID": 5},
  {"Task": "Add song to queue", "ID": 12},
  {"Task": "Play the queue", "ID": 19}
]
```

**Извлечение tool path:**

```python
tool_choice_ls = []
for task in task_path:
    if task["ID"] != -1 and task["ID"] in dic_tool:
        tool_choice_ls.append(dic_tool[task["ID"]]['tool_usage'])

# Output: ["search_track", "add_to_queue", "start_playback"]
```

**Отличия от других пайплайнов:**
- ✅ Специализирован для REST API
- ✅ ID инструментов в промте
- ❌ Нет фактического выполнения (только планирование)
- ❌ Нет parameter extraction (упрощенный вариант)

---

### 3.7 Сравнительная таблица всех EasyTool пайплайнов

| Аспект | FuncQA-MH | FuncQA-OH | ToolBench | ToolBench-RAG | RestBench |
|--------|-----------|-----------|-----------|---------------|-----------|
| **Декомпозиция** | ✅ 2 этапа | ❌ | ❌ | ❌ | ✅ 1 этап |
| **Топология зависимостей** | ✅ | ❌ | ❌ | ❌ | ❌ |
| **Tool Selection** | ✅ | ✅ | ✅ | ✅ | ❌ (ID в промте) |
| **API Selection** | ❌ | ❌ | ✅ | ✅ | ❌ |
| **Parameter Extraction** | ✅ | ✅ | ✅ | ✅ | ❌ |
| **Retrieval** | ❌ | ❌ | ❌ | ✅ Semantic | ❌ |
| **Execution** | ✅ Python funcs | ✅ Python funcs | ✅ Real APIs | ✅ Real APIs | ❌ Planning only |
| **Answer Generation** | ❌ | ❌ | ✅ Per API | ✅ Per API | ❌ |
| **Previous Log** | ✅ Cumulative | ❌ | ✅ Cumulative | ✅ Cumulative | ❌ |
| **Retry Mechanism** | ✅ Max 10 | ✅ Max 10 | ✅ Max 10 | ✅ Max 10 | ✅ Max 10 |

---

### 3.8 Данные между этапами: Детальный пример

**Сценарий:** ToolBench - "Book a hotel in Paris for 2 nights"

**Stage 1 Output (Tool Selection):**
```json
{"ID": 23}
```

**Tool Instruction Loading:**
```json
{
  "tool_id": 23,
  "tool_name": "Hotels API",
  "tool_description": "Search and book hotels worldwide",
  "apis": [
    {
      "name": "search_hotels",
      "description": "Search for hotels by location",
      "parameters": {
        "required": ["location"],
        "optional": ["check_in", "check_out", "guests"]
      }
    },
    {
      "name": "get_hotel_details",
      "description": "Get detailed information about a hotel",
      "parameters": {
        "required": ["hotel_id"]
      }
    },
    {
      "name": "book_hotel",
      "description": "Book a hotel room",
      "parameters": {
        "required": ["hotel_id", "check_in", "check_out"]
      }
    }
  ]
}
```

**Stage 2 Output (API Selection):**
```json
["search_hotels", "book_hotel"]
```

**API 1 Execution:**

*Stage 3 - Parameter Extraction:*
```json
{
  "Parameters": {
    "location": "Paris",
    "guests": 2
  }
}
```

*Stage 4 - API Call Result:*
```json
{
  "hotels": [
    {"id": "h_123", "name": "Paris Hotel", "price": 150},
    {"id": "h_456", "name": "Eiffel Tower Inn", "price": 200}
  ]
}
```

*Stage 5 - Answer Generation:*
```
"I found 2 hotels in Paris. The most affordable option is Paris Hotel at $150 per night (ID: h_123)."
```

*Previous Log Update:*
```
API: search_hotels
Question: Book a hotel in Paris for 2 nights
Parameters: {"location": "Paris", "guests": 2}
Result: {"hotels": [{"id": "h_123", "name": "Paris Hotel"...}]}
Answer: I found 2 hotels in Paris. The most affordable option is Paris Hotel at $150 per night (ID: h_123).
```

**API 2 Execution:**

*Stage 3 - Parameter Extraction with Dependencies:*
```python
# LLM видит previous_log и извлекает hotel_id
```

```json
{
  "Parameters": {
    "hotel_id": "h_123",
    "check_in": "2024-01-15",
    "check_out": "2024-01-17"
  }
}
```

*Stage 4 - API Call Result:*
```json
{
  "booking_id": "bk_789",
  "confirmation": "Your booking is confirmed",
  "total_price": 300
}
```

*Stage 5 - Final Answer:*
```
"Your hotel booking is confirmed! Booking ID: bk_789. You'll stay at Paris Hotel from Jan 15 to Jan 17. Total cost: $300 for 2 nights."
```

---

