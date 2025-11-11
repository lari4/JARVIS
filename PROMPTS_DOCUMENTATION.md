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

## 3. EasyTool - Система интеграции и оркестрации API инструментов

EasyTool - это framework для работы с API инструментами, который включает выбор инструментов, декомпозицию задач, извлечение параметров и генерацию ответов. Система работает с различными типами API (ToolBench, FuncQA, RestBench).

### 3.1 Tool Selection (Выбор инструмента)

**Расположение:** `easytool/easytool/toolbench.py` (строка 48-60) и `easytool/easytool/funcQA.py` (строка 48-61)

**Назначение:** Этот промт используется для выбора одного наиболее подходящего инструмента из списка доступных инструментов для решения пользовательского вопроса. AI должен проанализировать вопрос и выбрать единственный инструмент, который лучше всего подходит для решения задачи.

**Ключевые особенности:**
- Выбор только одного инструмента из списка
- Выход в формате JSON с ID инструмента
- Использование Tool List с ID и описаниями

```python
template = "You are a helpful assistant."
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "This is the user's question: {question}\n"
    "These are the tools you can select to solve the question:\n"
    "Tool List:\n"
    "{Too_list}\n\n"
    "Please note that: \n"
    "1. You should only chooce one tool the Tool List to solve this question.\n"
    "2. You must ONLY output the ID of the tool you chose in a parsible JSON format. Two example outputs look like:\n"
    "'''\n"
    "Example 1: {{\"ID\": 1}}\n"
    "Example 2: {{\"ID\": 2}}\n"
    "'''\n"
    "Output:"
)
```

### 3.2 API Selection (Выбор API)

**Расположение:** `easytool/easytool/toolbench.py` (строка 84-108)

**Назначение:** После выбора инструмента, этот промт используется для выбора конкретных API из инструкции к инструменту. AI может выбрать несколько API для решения вопроса.

**Ключевые особенности:**
- Выбор нескольких API из списка
- Использование API instruction для контекста
- Выход в формате Python List

```python
input_execute_rapidapi_api_note = '''
This is an API Tool instruction. Given a question, you should choose APIs from the API list you want to use for this question in this instruction.
you must only output in a parsible Python List Format. An example output looks like:
```
["api1", "api2", ...]
```
'''.strip()

human_message_prompt = HumanMessagePromptTemplate.from_template(
    "{API_instruction}\n"
    "{input_execute_rapidapi_api_note}\n"
    "This is the API list: {API_list}\n"
    "Please note that: \n"
    "1. The APIs you choose must in the API list.\n"
    "2. You must ONLY output in the following parsible Python List Format.\n"
    "```\n"
    "Output_Example: [\"api1\", \"api2\", ...]\n"
    "```\n"
    "Question: {question}\n"
    "Output:"
)
```

### 3.3 Parameter Extraction (Извлечение параметров)

**Расположение:** `easytool/easytool/toolbench.py` (строка 139-158) и `easytool/easytool/funcQA.py` (строка 195-214)

**Назначение:** Этот промт используется для извлечения параметров API из вопроса пользователя на основе документации API. AI должен сгенерировать корректные параметры для вызова API.

**Ключевые особенности:**
- Генерация required и optional параметров
- Поддержка множественных вызовов API (параметры в виде списка)
- Использование примеров из документации
- Выход в формате JSON

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "This is an API tool documentation. Given a user's question, you need to output parameters according to the API tool documentation to successfully call the API to solve the user's question.\n"
    "This is API tool documentation: {api_dic}\n"
    "Please note that: \n"
    "1. The Example in the API tool documentation can help you better understand the use of the API.\n"
    "2. Ensure the parameters you output are correct. The output must contain the required parameters, and can contain the optional parameters based on the question. If no paremters in the required parameters and optional parameters, just leave it as {{\"Parameters\":{{}}}}\n"
    "3. If the user's question mentions other APIs, you should ONLY consider the API tool documentation I give and do not consider other APIs.\n"
    "4. If you need to use this API multiple times, please set \"Parameters\" to a list.\n"
    "5. You must ONLY output in a parsible JSON format. Two examples output looks like:\n"
    "'''\n"
    "Example 1: {{\"Parameters\":{{\"keyword\": \"Artificial Intelligence\", \"language\": \"English\"}}}}\n"
    "Example 2: {{\"Parameters\":[{{\"keyword\": \"Artificial Intelligence\", \"language\": \"English\"}}, {{\"keyword\": \"Machine Learning\", \"language\": \"English\"}}]}}\n"
    "'''\n"
    "This is user's question: {question}\n"
    "Output:\n"
)
```

### 3.4 Parameter Extraction with Dependencies (Извлечение параметров с зависимостями)

**Расположение:** `easytool/easytool/toolbench.py` (строка 181-203) и `easytool/easytool/funcQA.py` (строка 237-258)

**Назначение:** Расширенная версия промта для извлечения параметров, которая учитывает зависимости от предыдущих вызовов API. AI может использовать результаты предыдущих API для формирования параметров текущего API.

**Ключевые особенности:**
- Использование логов предыдущих вопросов и ответов
- Поддержка зависимостей между последовательными API вызовами
- Контекстно-зависимая генерация параметров

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "Given a user's question and a API tool documentation, you need to output parameters according to the API tool documentation to successfully call the API to solve the user's question.\n"
    "Please note that: \n"
    "1. The Example in the API tool documentation can help you better understand the use of the API.\n"
    "2. Ensure the parameters you output are correct. The output must contain the required parameters, and can contain the optional parameters based on the question. If no paremters in the required parameters and optional parameters, just leave it as {{\"Parameters\":{{}}}}\n"
    "3. If the user's question mentions other APIs, you should ONLY consider the API tool documentation I give and do not consider other APIs.\n"
    "4. The question may have dependencies on answers of other questions, so we will provide logs of previous questions and answers for your reference.\n"
    "5. If you need to use this API multiple times,, please set \"Parameters\" to a list.\n"
    "6. You must ONLY output in a parsible JSON format. Two examples output looks like:\n"
    "'''\n"
    "Example 1: {{\"Parameters\":{{\"keyword\": \"Artificial Intelligence\", \"language\": \"English\"}}}}\n"
    "Example 2: {{\"Parameters\":[{{\"keyword\": \"Artificial Intelligence\", \"language\": \"English\"}}, {{\"keyword\": \"Machine Learning\", \"language\": \"English\"}}]}}\n"
    "'''\n"
    "There are logs of previous questions and answers: \n {previous_log}\n"
    "This is the current user's question: {question}\n"
    "This is API tool documentation: {api_dic}\n"
    "Output:\n"
)
```

### 3.5 Task Decomposition (Декомпозиция задач)

**Расположение:** `easytool/easytool/funcQA.py` (строка 88-106)

**Назначение:** Этот промт используется для разбиения сложного вопроса пользователя на простые подзадачи, каждая из которых может быть выполнена одним инструментом.

**Ключевые особенности:**
- Разбиение на простые подзадачи
- Указание зависимостей между задачами (использование результатов X, Y)
- Каждая подзадача выполняется одним инструментом

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "You need to decompose a complex user's question into some simple subtasks and let the model execute it step by step.\n"
    "This is the user's question: {question}\n"
    "This is tool list:\n"
    "{Tool_list}\n"
    "Please note that: \n"
    "1. You should only decompose this complex user's question into some simple subtasks which can be executed easily by using one single tool in the tool list.\n"
    "2. If one subtask need the results from other subtask, you can should write clearly. For example:"
    "{{\"Tasks\": [\"Convert 23 km/h to X km/min by 'divide_'\", \"Multiply X km/min by 45 min to get Y by 'multiply_'\"]}}\n"
    "3. You must ONLY output in a parsible JSON format. An example output looks like:\n"
    "'''\n"
    "{{\"Tasks\": [\"Task 1\", \"Task 2\", ...]}}\n"
    "'''\n"
    "Output:"
)
```

### 3.6 Task Topology (Топология задач)

**Расположение:** `easytool/easytool/funcQA.py` (строка 128-145)

**Назначение:** Этот промт используется для определения логических связей и порядка выполнения подзадач, включая зависимости между ними.

**Ключевые особенности:**
- Определение зависимостей между задачами
- Построение графа выполнения задач
- Идентификация задач без зависимостей (dep: -1)

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "Given a complex user's question, I have decompose this question into some simple subtasks"
    "I think there exists a logical connections and order amontg the tasks. "
    "Thus you need to help me output this logical connections and order.\n"
    "You must ONLY output in a parsible JSON format with the following format:\n"
    "'''\n"
    "[{{\"task\": task, \"id\", task_id, \"dep\": [dependency_task_id1, dependency_task_id2, ...]}}]\n"
    "'''\n"
    "The \"dep\" field denotes the id of the previous task which generates a new resource upon which the current task depends. If there are no dependencies, set \"dep\" to -1.\n\n"
    "This is user's question: {question}\n"
    "These are subtasks of this question:\n"
    "{task_ls}\n"
    "Output: "
)
```

### 3.7 Task Decomposition for RestBench (Декомпозиция для REST API)

**Расположение:** `easytool/easytool/restbench.py` (строка 46-66)

**Назначение:** Специализированный промт для декомпозиции задач при работе с REST API (например, Spotify API). Включает выбор конкретных инструментов по ID для каждой подзадачи.

**Ключевые особенности:**
- Работа с базой данных (Spotify)
- Указание ID инструмента для каждой подзадачи
- Учет логических связей, порядка и ограничений между инструментами
- Поддержка задач без инструментов (ID: -1)

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "We have spotify database and the following tools:\n"
    "{Tool_dic}"
    "You need to decompose a complex user's question into some simple subtasks and let the model execute it step by step with these tools.\n"
    "Please note that: \n"
    "1. you should break down tasks into appropriate subtasks to use the tools mentioned above.\n"
    "2. You should not only list the subtask, but also list the ID of the tool used to solve this subtask.\n"
    "3. If you think you do not need to use the tool to solve the subtask, just leave it as {{\"ID\": -1}}\n"
    "4. You must consider the logical connections, order and constraints among the tools to achieve a correct tool path."
    "5. You must ONLY output the ID of the tool you chose in a parsible JSON format. Two examples output look like:\n"
    "'''\n"
    "Question: Pause the player"
    "Example 1: [{{\"Task\":\"Get information about the user's current playback state\", \"ID\":15}}, {{\"Task\":\"Pause playback on the user's account\", \"ID\":19}}]\n"
    "'''\n"
    "This is the user's question: {question}\n"
    "Output:"
)
```

### 3.8 Answer Generation (Генерация ответа)

**Расположение:** `easytool/easytool/toolbench.py` (строка 226-240)

**Назначение:** Этот промт используется для генерации естественного языкового ответа на основе результата вызова API. AI должен преобразовать технический вывод API в понятный пользователю ответ.

**Ключевые особенности:**
- Преобразование API response в естественный язык
- Детальное использование информации из ответа
- Пользователь не видит прямой ответ API

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "You should answer the question based on the response output by the API tool."
    "Please note that:\n"
    "1. Answer the question in natural language based on the API response reasonably and effectively.\n"
    "2. The user cannot directly get API response, "
    "so you need to make full use of the response and give the information "
    "in the response that can satisfy the user's question in as much detail as possible.\n"
    "This is the user's question:\n {question}\n"
    "This is the API response:\n {call_result}\n"
    "Output:"
)
```

### 3.9 Answer Generation with Dependencies (Генерация ответа с зависимостями)

**Расположение:** `easytool/easytool/toolbench.py` (строка 258-277)

**Назначение:** Расширенная версия промта для генерации ответа, которая учитывает зависимости от предыдущих вопросов и ответов.

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "You should answer the question based on the response output by the API tool."
    "Please note that:\n"
    "1. Try to organize the response into a natural language answer.\n"
    "2. We will not show the API response to the user, "
    "thus you need to make full use of the response and give the information "
    "in the response that can satisfy the user's question in as much detail as possible.\n"
    "3. The question may have dependencies on answers of other questions, so we will provide logs of previous questions and answers.\n"
    "There are logs of previous questions and answers: \n {previous_log}\n"
    "This is the user's question: {question}\n"
    "This is the response output by the API tool: \n{call_result}\n"
    "We will not show the API response to the user, "
    "thus you need to make full use of the response and give the information "
    "in the response that can satisfy the user's question in as much detail as possible.\n"
    "Output:"
)
```

### 3.10 Direct Answer Generation (Прямая генерация ответа без API)

**Расположение:** `easytool/easytool/funcQA.py` (строка 180-192)

**Назначение:** Fallback промт для случаев, когда вопрос можно ответить напрямую без использования инструментов.

**Ключевые особенности:**
- Простой прямой ответ на вопрос
- Не требует вызова API
- Используется для общих вопросов

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "You need to answer the user's question.\n"
    "This is the user's question: {task}\n"
    "Output:"
)
```

### 3.11 Answer Verification (Проверка ответа)

**Расположение:** `easytool/easytool/toolbench.py` (строка 296-316)

**Назначение:** Этот промт используется для проверки, может ли сгенерированный ответ разумно и точно ответить на вопрос пользователя.

**Ключевые особенности:**
- Валидация качества ответа
- Обоснование решения (Reason)
- Бинарный выбор: Yes/No

```python
human_message_prompt = HumanMessagePromptTemplate.from_template(
    "Please check whether the response can reasonably and accurately answer the question."
    "If can, please output 'YES'; If not, please output 'NO'\n"
    "You need to give reasons first and then decide whether the response can reasonably and accurately answer the question. You must only output in a parsible JSON format. Two example outputs look like:\n"
    "Example 1: {{\"Reason\": \"The reason why you think the response can reasonably and accurately answer the question\", \"Choice\": \"Yes\"}}\n"
    "Example 2: {{\"Reason\": \"The reason why you think the response cannot reasonably and accurately answer the question\", \"Choice\": \"No\"}}\n"
    "This is the user's question: {question}\n"
    "This is the response: {answer}\n"
    "Output: "
)
```

---

