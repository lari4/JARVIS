# Документация AI-промтов проекта JARVIS

Этот документ содержит полное описание всех AI-промтов, используемых в проекте JARVIS. Промты сгруппированы по модулям и назначению.

---

## 1. HuggingGPT - Мультимодальная AI система

HuggingGPT - это система для разбора пользовательских запросов, выбора подходящих моделей и генерации ответов. Система работает в три этапа.

### 1.1 Task Parsing (Разбор задач)

**Расположение:** `hugginggpt/server/configs/config.default.yaml` (строка 31-32)

**Назначение:** Этот промт используется на первом этапе работы HuggingGPT для разбора пользовательского ввода на структурированные задачи. AI должен определить, какие типы задач нужно выполнить, в каком порядке, и какие зависимости существуют между задачами. Промт поддерживает 34+ типов задач включая NLP, Computer Vision, Audio и мультимодальные задачи.

**Ключевые особенности:**
- Определение зависимостей между задачами через поле "dep"
- Использование специального тега `<GENERATED>-dep_id` для ссылки на ресурсы из предыдущих задач
- Поддержка трех типов аргументов: text, image, audio
- Выход в формате JSON

```yaml
tprompt:
  parse_task: >-
    #1 Task Planning Stage: The AI assistant can parse user input to several tasks: [{"task": task, "id": task_id, "dep": dependency_task_id, "args": {"text": text or <GENERATED>-dep_id, "image": image_url or <GENERATED>-dep_id, "audio": audio_url or <GENERATED>-dep_id}}]. The special tag "<GENERATED>-dep_id" refer to the one generated text/image/audio in the dependency task (Please consider whether the dependency task generates resources of this type.) and "dep_id" must be in "dep" list. The "dep" field denotes the ids of the previous prerequisite tasks which generate a new resource that the current task relies on. The "args" field must in ["text", "image", "audio"], nothing else. The task MUST be selected from the following options: "token-classification", "text2text-generation", "summarization", "translation", "question-answering", "conversational", "text-generation", "sentence-similarity", "tabular-classification", "object-detection", "image-classification", "image-to-image", "image-to-text", "text-to-image", "text-to-video", "visual-question-answering", "document-question-answering", "image-segmentation", "depth-estimation", "text-to-speech", "automatic-speech-recognition", "audio-to-audio", "audio-classification", "canny-control", "hed-control", "mlsd-control", "normal-control", "openpose-control", "canny-text-to-image", "depth-text-to-image", "hed-text-to-image", "mlsd-text-to-image", "normal-text-to-image", "openpose-text-to-image", "seg-text-to-image". There may be multiple tasks of the same type. Think step by step about all the tasks needed to resolve the user's request. Parse out as few tasks as possible while ensuring that the user request can be resolved. Pay attention to the dependencies and order among tasks. If the user input can't be parsed, you need to reply empty JSON [].
```

**Дополнительный промт для контекста:**

```yaml
prompt:
  parse_task: The chat log [ {{context}} ] may contain the resources I mentioned. Now I input { {{input}} }. Pay attention to the input and output types of tasks and the dependencies between tasks.
```

### 1.2 Few-Shot Examples для Task Parsing

**Расположение:** `hugginggpt/server/demos/demo_parse_task.json`

**Назначение:** Этот файл содержит примеры правильного разбора пользовательских запросов на структурированные задачи. Используется для few-shot learning - AI видит эти примеры перед тем, как разбирать новый запрос пользователя.

**Примеры включают:**
- Подсчет овец на нескольких изображениях
- Детекция объектов с генерацией похожих изображений и видео
- Ответы на вопросы по документам с голосовым выводом
- Генерация изображений с использованием ControlNet
- Мультимодальную генерацию (видео + изображение + озвучка)

```json
[
    {
        "role": "user",
        "content": "Give you some pictures e1.jpg, e2.png, e3.jpg, help me count the number of sheep?"
    },
    {
        "role": "assistant",
        "content": "[{\"task\": \"image-to-text\", \"id\": 0, \"dep\": [-1], \"args\": {\"image\": \"e1.jpg\" }}, {\"task\": \"object-detection\", \"id\": 1, \"dep\": [-1], \"args\": {\"image\": \"e1.jpg\" }}, {\"task\": \"visual-question-answering\", \"id\": 2, \"dep\": [1], \"args\": {\"image\": \"<GENERATED>-1\", \"text\": \"How many sheep in the picture\"}} }}, {\"task\": \"image-to-text\", \"id\": 3, \"dep\": [-1], \"args\": {\"image\": \"e2.png\" }}, {\"task\": \"object-detection\", \"id\": 4, \"dep\": [-1], \"args\": {\"image\": \"e2.png\" }}, {\"task\": \"visual-question-answering\", \"id\": 5, \"dep\": [4], \"args\": {\"image\": \"<GENERATED>-4\", \"text\": \"How many sheep in the picture\"}} }}, {\"task\": \"image-to-text\", \"id\": 6, \"dep\": [-1], \"args\": {\"image\": \"e3.jpg\" }},  {\"task\": \"object-detection\", \"id\": 7, \"dep\": [-1], \"args\": {\"image\": \"e3.jpg\" }}, {\"task\": \"visual-question-answering\", \"id\": 8, \"dep\": [7], \"args\": {\"image\": \"<GENERATED>-7\", \"text\": \"How many sheep in the picture\"}}]"
    },

    {
        "role":"user",
        "content":"Look at /e.jpg, can you tell me how many objects in the picture? Give me a picture and video similar to this one."
    },
    {
        "role":"assistant",
        "content":"[{\"task\": \"image-to-text\", \"id\": 0, \"dep\": [-1], \"args\": {\"image\": \"/e.jpg\" }}, {\"task\": \"object-detection\", \"id\": 1, \"dep\": [-1], \"args\": {\"image\": \"/e.jpg\" }}, {\"task\": \"visual-question-answering\", \"id\": 2, \"dep\": [1], \"args\": {\"image\": \"<GENERATED>-1\", \"text\": \"how many objects in the picture?\" }}, {\"task\": \"text-to-image\", \"id\": 3, \"dep\": [0], \"args\": {\"text\": \"<GENERATED-0>\" }}, {\"task\": \"image-to-image\", \"id\": 4, \"dep\": [-1], \"args\": {\"image\": \"/e.jpg\" }}, {\"task\": \"text-to-video\", \"id\": 5, \"dep\": [0], \"args\": {\"text\": \"<GENERATED-0>\" }}]"
    },

    {
        "role":"user",
        "content":"given a document /images/e.jpeg, answer me what is the student amount? And describe the image with your voice"
    },
    {
        "role":"assistant",
        "content":"{\"task\": \"document-question-answering\", \"id\": 0, \"dep\": [-1], \"args\": {\"image\": \"/images/e.jpeg\", \"text\": \"what is the student amount?\" }}, {\"task\": \"visual-question-answering\", \"id\": 1, \"dep\": [-1], \"args\": {\"image\": \"/images/e.jpeg\", \"text\": \"what is the student amount?\" }}, {\"task\": \"image-to-text\", \"id\": 2, \"dep\": [-1], \"args\": {\"image\": \"/images/e.jpg\" }}, {\"task\": \"text-to-speech\", \"id\": 3, \"dep\": [2], \"args\": {\"text\": \"<GENERATED>-2\" }}]"
    },

    {
        "role": "user",
        "content": "Given an image /example.jpg, first generate a hed image, then based on the hed image generate a new image where a girl is reading a book"
    },
    {
        "role": "assistant",
        "content": "[{\"task\": \"openpose-control\", \"id\": 0, \"dep\": [-1], \"args\": {\"image\": \"/example.jpg\" }},  {\"task\": \"openpose-text-to-image\", \"id\": 1, \"dep\": [0], \"args\": {\"text\": \"a girl is reading a book\", \"image\": \"<GENERATED>-0\" }}]"
    },

    {
        "role": "user",
        "content": "please show me a video and an image of (based on the text) 'a boy is running' and dub it"
    },
    {
        "role": "assistant",
        "content": "[{\"task\": \"text-to-video\", \"id\": 0, \"dep\": [-1], \"args\": {\"text\": \"a boy is running\" }}, {\"task\": \"text-to-speech\", \"id\": 1, \"dep\": [-1], \"args\": {\"text\": \"a boy is running\" }}, {\"task\": \"text-to-image\", \"id\": 2, \"dep\": [-1], \"args\": {\"text\": \"a boy is running\" }}]"
    },


    {
        "role": "user",
        "content": "please show me a joke and an image of cat"
    },
    {
        "role": "assistant",
        "content": "[{\"task\": \"conversational\", \"id\": 0, \"dep\": [-1], \"args\": {\"text\": \"please show me a joke of cat\" }}, {\"task\": \"text-to-image\", \"id\": 1, \"dep\": [-1], \"args\": {\"text\": \"a photo of cat\" }}]"
    }
]
```

### 1.3 Model Selection (Выбор моделей)

**Расположение:** `hugginggpt/server/configs/config.default.yaml` (строка 33-34)

**Назначение:** На втором этапе работы HuggingGPT AI должен выбрать наиболее подходящую модель из списка доступных моделей для каждой задачи. Промт инструктирует AI фокусироваться на описании моделей и находить модель с наибольшим потенциалом для решения задачи. Предпочтение отдается моделям с локальными inference endpoints для скорости и стабильности.

**Ключевые особенности:**
- Выбор на основе описания модели и требований задачи
- Предпочтение локальным endpoints для производительности
- Выход в формате JSON с обоснованием выбора

```yaml
tprompt:
  choose_model: >-
    #2 Model Selection Stage: Given the user request and the parsed tasks, the AI assistant helps the user to select a suitable model from a list of models to process the user request. The assistant should focus more on the description of the model and find the model that has the most potential to solve requests and tasks. Also, prefer models with local inference endpoints for speed and stability.
```

**Дополнительный промт для конкретного выбора:**

```yaml
prompt:
  choose_model: >-
    Please choose the most suitable model from {{metas}} for the task {{task}}. The output must be in a strict JSON format: {"id": "id", "reason": "your detail reasons for the choice"}.
```

**Шаблон для Few-Shot примеров:**

**Расположение:** `hugginggpt/server/demos/demo_choose_model.json`

```json
[
    {
        "role": "user",
        "content": "{{input}}"
    },
    {
        "role": "assistant",
        "content": "{{task}}"
    }
]
```

Переменные {{input}} и {{task}} динамически заполняются пользовательским запросом и метаданными моделей во время выполнения.

### 1.4 Response Generation (Генерация ответа)

**Расположение:** `hugginggpt/server/configs/config.default.yaml` (строка 35-36)

**Назначение:** На финальном четвертом этапе (stage #4) AI должен проанализировать логи выполнения задач и создать человеко-понятный ответ. AI должен описать процесс выполнения задач и результаты инференса, фильтруя нерелевантную информацию и предоставляя полные пути к файлам или URL результатов.

**Ключевые особенности:**
- Анализ логов выполнения и создание дружественного ответа
- Описание рабочего процесса (workflow) включая использованные модели
- Фильтрация нерелевантной информации
- Предоставление полных путей/URL к результатам
- Обработка случаев, когда инференс не дал результатов

```yaml
tprompt:
  response_results: >-
    #4 Response Generation Stage: With the task execution logs, the AI assistant needs to describe the process and inference results.
```

**Расширенный промт с инструкциями:**

```yaml
prompt:
  response_results: >-
    Yes. Please first think carefully and directly answer my request based on the inference results. Some of the inferences may not always turn out to be correct and require you to make careful consideration in making decisions. Then please detail your workflow including the used models and inference results for my request in your friendly tone. Please filter out information that is not relevant to my request. Tell me the complete path or urls of files in inference results. If there is nothing in the results, please tell me you can't make it.
```

**Шаблон для диалога:**

**Расположение:** `hugginggpt/server/demos/demo_response_results.json`

```json
[
    {
        "role": "user",
        "content": "{{input}}"
    },
    {
        "role": "assistant",
        "content": "Before give you a response, I want to introduce my workflow for your request, which is shown in the following JSON data: {{processes}}. Do you have any demands regarding my response?"
    }
]
```

Переменная {{processes}} заполняется информацией о выполненных задачах и использованных моделях.

---

## 2. TaskBench - Система планирования и декомпозиции задач

TaskBench - это benchmark система для тестирования способностей LLM декомпозировать сложные задачи на простые шаги с использованием различных инструментов. Поддерживает два типа зависимостей: resource dependency (зависимость по ресурсам) и temporal dependency (временная зависимость).

### 2.1 Task Planning - Resource Dependency (Планирование с зависимостями по ресурсам)

**Расположение:** `taskbench/inference.py` (строка 175-176)

**Назначение:** Этот промт используется для генерации шагов задачи и узлов задачи с зависимостями по ресурсам. AI должен разбить пользовательский запрос на последовательные шаги, где выходные данные одного шага могут использоваться как входные данные для другого. Используется для данных HuggingFace и мультимедиа.

**Ключевые особенности:**
- Строгий формат JSON вывода с task_steps и task_nodes
- Использование тегов `<node-j>` для ссылки на выход j-го узла
- Проверка соответствия типов входных данных (input-type) и выходных данных (output-type)
- Минимизация количества шагов при полном решении запроса

```python
prompt = """\n# GOAL #: Based on the above tools, I want you generate task steps and task nodes to solve the # USER REQUEST #. The format must in a strict JSON format, like: {"task_steps": [ step description of one or more steps ], "task_nodes": [{"task": "tool name must be from # TOOL LIST #", "arguments": [ a concise list of arguments for the tool. Either original text, or user-mentioned filename, or tag '<node-j>' (start from 0) to refer to the output of the j-th node. ]}]} """

prompt += """\n\n# REQUIREMENTS #: \n1. the generated task steps and task nodes can resolve the given user request # USER REQUEST # perfectly. Task name must be selected from # TASK LIST #; \n2. the task steps should strictly aligned with the task nodes, and the number of task steps should be same with the task nodes; \n3. the dependencies among task steps should align with the argument dependencies of the task nodes; \n4. the tool arguments should be align with the input-type field of # TASK LIST #;"""
```

**Пример использования с демо-данными:**

```python
prompt += f"""\n# EXAMPLE #:\n# USER REQUEST #: {demo["user_request"]}\n# RESULT #: {json.dumps(demo["result"])}"""
```

**Финальная часть промта:**

```python
prompt += """\n\n# USER REQUEST #: {{user_request}}\nnow please generate your result in a strict JSON format:\n# RESULT #:"""
```

### 2.2 Task Planning - Temporal Dependency (Планирование с временными зависимостями)

**Расположение:** `taskbench/inference.py` (строка 178-179)

**Назначение:** Этот промт используется для генерации шагов задачи с временными зависимостями. В отличие от resource dependency, здесь явно указывается порядок вызова API через поле task_links. Используется для данных Daily Life APIs, где важна последовательность выполнения операций.

**Ключевые особенности:**
- Включает поле task_links для указания временной последовательности
- Формат шагов: "Step x: Call xxx tool with xxx: 'xxx' and xxx: 'xxx'"
- Аргументы с именем и значением параметра
- Явная спецификация зависимостей источник->цель

```python
prompt = """\n# GOAL #:\nBased on the above tools, I want you generate task steps and task nodes to solve the # USER REQUEST #. The format must in a strict JSON format, like: {"task_steps": [ "concrete steps, format as Step x: Call xxx tool with xxx: 'xxx' and xxx: 'xxx'" ], "task_nodes": [{"task": "task name must be from # TASK LIST #", "arguments": [ {"name": "parameter name", "value": "parameter value, either user-specified text or the specific name of the tool whose result is required by this node"} ]}], "task_links": [{"source": "task name i", "target": "task name j"}]}"""

prompt += """\n\n# REQUIREMENTS #: \n1. the generated task steps and task nodes can resolve the given user request # USER REQUEST # perfectly. Task name must be selected from # TASK LIST #; \n2. the task steps should strictly aligned with the task nodes, and the number of task steps should be same with the task nodes; \n3. The task links (task_links) should reflect the temporal dependencies among task nodes, i.e. the order in which the APIs are invoked;"""
```

### 2.3 JSON Reformatting - Resource Dependency (Переформатирование для Resource)

**Расположение:** `taskbench/inference.py` (строка 251)

**Назначение:** Когда LLM генерирует некорректный JSON, этот промт используется для исправления формата без изменения смысла. Это recovery mechanism для обработки ошибок парсинга.

**Ключевые особенности:**
- Исправление формата без изменения смысла
- Гарантия совместимости с json.loads()
- Сохранение исходной структуры задачи

```python
prompt = """Please format the result # RESULT # to a strict JSON format # STRICT JSON FORMAT #. \nRequirements:\n1. Do not change the meaning of task steps and task nodes;\n2. Don't tolerate any possible irregular formatting to ensure that the generated content can be converted by json.loads();\n3. You must output the result in this schema: {"task_steps": [ step description of one or more steps ], "task_nodes": [{"task": "tool name must be from # TOOL LIST #", "arguments": [ a concise list of arguments for the tool. Either original text, or user-mentioned filename, or tag '<node-j>' (start from 0) to refer to the output of the j-th node. ]}]}\n# RESULT #:{{illegal_result}}\n# STRICT JSON FORMAT #:"""
```

### 2.4 JSON Reformatting - Temporal Dependency (Переформатирование для Temporal)

**Расположение:** `taskbench/inference.py` (строка 253)

**Назначение:** Аналогичен промту для resource dependency, но включает дополнительную обработку task_links.

```python
prompt = """Please format the result # RESULT # to a strict JSON format # STRICT JSON FORMAT #. \nRequirements:\n1. Do not change the meaning of task steps, task nodes and task links;\n2. Don't tolerate any possible irregular formatting to ensure that the generated content can be converted by json.loads();\n3. Pay attention to the matching of brackets. Write in a compact format and avoid using too many space formatting controls;\n4. You must output the result in this schema: {"task_steps": [ "concrete steps, format as Step x: Call xxx tool with xxx: 'xxx' and xxx: 'xxx'" ], "task_nodes": [{"task": "task name must be from # TASK LIST #", "arguments": [ {"name": "parameter name", "value": "parameter value, either user-specified text or the specific name of the tool whose result is required by this node"} ]}], "task_links": [{"source": "task name i", "target": "task name j"}]}\n# RESULT #:{{illegal_result}}\n# STRICT JSON FORMAT #:"""
```

---

