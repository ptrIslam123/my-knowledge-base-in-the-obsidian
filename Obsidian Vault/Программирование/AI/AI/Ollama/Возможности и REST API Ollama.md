### **1. Текстовые возможности**
- ✅ **Понимание и генерация текста** на 100+ языках
- ✅ **Многоязычность**: отличное понимание русского, английского, китайского
- ✅ **Поэзия и творческое письмо**
- ✅ **Резюме и перефразирование** текстов
- ✅ **Перевод** между языками
- ✅ **Анализ тональности** и эмоций
- ✅ **Извлечение ключевых слов** и сущностей

### **2. Программирование и кодирование**
- ✅ **Генерация кода** на Python, JavaScript, Java, C++, Go, Rust и др.
- ✅ **Отладка и объяснение кода**
- ✅ **Рефакторинг и оптимизация**
- ✅ **Документирование кода**
- ✅ **Генерация SQL-запросов**
- ✅ **Shell-скрипты** (Bash, PowerShell)
- ✅ **Конвертация** между языками программирования
- ✅ **Решение алгоритмических задач**

### **3. Математика и логика**
- ✅ **Решение математических задач**
- ✅ **Логические рассуждения**
- ✅ **Статистический анализ**
- ✅ **Построение алгоритмов**
- ✅ **Работа с формулами**

### **4. Анализ данных**
- ✅ **Обработка CSV/JSON данных**
- ✅ **Генерация аналитических отчетов**
- ✅ **Визуализация данных** (описание графиков)
- ✅ **Предсказательная аналитика**

### **5. Работа с документами**
- ✅ **Анализ и суммаризация PDF/Word**
- ✅ **Извлечение структурированной информации**
- ✅ **Сравнение документов**
- ✅ **Генерация шаблонов документов**

## **Технические возможности API Ollama для Qwen**

### **Параметры генерации:**
```python
{
    "model": "qwen3:4b",
    "prompt": "текст",
    "stream": True/False,           # Стриминг ответов
    "options": {
        "temperature": 0.7,        # Креативность (0-2)
        "top_p": 0.9,             # Качество генерации
        "top_k": 40,              # Разнообразие
        "num_predict": 512,       # Макс. токенов в ответе
        "repeat_penalty": 1.1,    # Штраф за повторения
        "presence_penalty": 0.0,  # Штраф за повтор тем
        "frequency_penalty": 0.0, # Штраф за частые слова
        "mirostat": 2,            # Продвинутый контроль
        "seed": 42,               # Seed для воспроизводимости
        "stop": ["\n", "###"],    # Стоп-последовательности
        "num_ctx": 4096,          # Размер контекста
        "num_batch": 512,         # Размер батча
        "num_thread": 4,          # Количество потоков
        "main_gpu": 0,            # GPU для вычислений
        "tfs_z": 1.0              # Tail free sampling
    },
    "template": "{{ .Prompt }}",  # Шаблон промпта
    "raw": True,                  # Сырой режим без шаблона
    "format": "json",             # Формат ответа
    "keep_alive": "5m"           # Удержание модели в памяти
}
```

### **Доступные эндпоинты:**

```python
# 1. Генерация текста
POST /api/generate

# 2. Чат с историей (поддерживает контекст)
POST /api/chat

# 3. Создание эмбеддингов
POST /api/embeddings

# 4. Получение информации о моделях
GET /api/tags

# 5. Показать детали модели
POST /api/show

# 6. Скачать модель
POST /api/pull

# 7. Удалить модель
DELETE /api/delete

# 8. Копировать модель
POST /api/copy

# 9. Получить информацию о сервере
GET /api/version
```

## **Практические применения Qwen API**

### **Пример 1: Анализ настроений**
```python
def analyze_sentiment(text):
    prompt = f"""Проанализируй тональность текста:
    
Текст: "{text}"

Ответь в формате JSON:
{{
    "sentiment": "positive/negative/neutral",
    "confidence": 0-1,
    "key_phrases": ["список", "фраз"],
    "emotions": ["эмоции"]
}}"""
    
    return ask_qwen(prompt, format="json")
```

### **Пример 2: Генератор документации**
```python
def generate_docs(code):
    prompt = f"""Создай документацию для этого кода:
python
{code}

Включи:
1. Описание функции
2. Параметры
3. Возвращаемое значение
4. Примеры использования
5. Исключения"""
    
    return ask_qwen(prompt)
```

### **Пример 3: SQL-ассистент**
```python
def generate_sql_query(natural_language, schema):
    prompt = f"""Создай SQL запрос на основе описания.

Схема таблиц:
{schema}

Запрос: {natural_language}

Только SQL код без объяснений:"""
    
    return ask_qwen(prompt)
```

---
##  **Специфические возможности Qwen 2.5**

### **Особенности Qwen 2.5 (последние версии):**
- ✅ **Улучшенное понимание контекста** (до 128K токенов)
- ✅ **Поддержка инструментов** (function calling)
- ✅ **Понимание структурированных данных**
- ✅ **Работа с таблицами и CSV**
- ✅ **Генерация маркдаун** с таблицами и формулами
- ✅ **Понимание временных рядов**
- ✅ **Мультимодальность** (в Qwen2.5-VL версиях)

### **Function Calling (имитация):**
```python
def qwen_with_tools(prompt, available_tools):
    system_prompt = f"""У тебя есть доступ к инструментам:
{available_tools}

Если пользователь просит что-то, что требует инструмента, ответь в формате:
TOOL_CALL: {{"tool": "имя_инструмента", "params": {{"param": "value"}}}}

Иначе отвечай нормально."""
    
    return ask_qwen(prompt, system=system_prompt)
```

## 🔌 **Интеграции и плагины**

### **Совместимость:**
- ✅ **OpenAI-совместимый API** (можно использовать с библиотеками для OpenAI)
- ✅ **LangChain** интеграция
- ✅ **LlamaIndex** поддержка
- ✅ **Semantic Kernel**
- ✅ **AutoGen** агенты

### **Пример LangChain:**
```python
from langchain_community.llms import Ollama
from langchain.chains import LLMChain
from langchain.prompts import PromptTemplate

llm = Ollama(model="qwen3:4b", temperature=0.7)

prompt = PromptTemplate(
    input_variables=["topic"],
    template="Расскажи о {topic} подробно"
)

chain = LLMChain(llm=llm, prompt=prompt)
result = chain.run("искусственный интеллект")
```

## 🛡️ **Безопасность и модерация**

### **Встроенные возможности:**
- ✅ **Фильтрация вредоносного контента**
- ✅ **Этическая генерация**
- ✅ **Предотвращение jailbreak**
- ✅ **Контроль тематики**

### **Кастомная модерация:**
```python
def safe_generate(prompt):
    # Проверка на недопустимые запросы
    banned_keywords = ["взлом", "оружие", "ненависть"]
    
    if any(keyword in prompt.lower() for keyword in banned_keywords):
        return "Запрос отклонен по соображениям безопасности"
    
    return ask_qwen(prompt)
```

## 📈 **Мониторинг и логирование**

```python
class QwenAPIMonitor:
    def __init__(self):
        self.stats = {
            "requests": 0,
            "tokens_generated": 0,
            "avg_response_time": 0,
            "errors": 0
        }
    
    def log_request(self, prompt, response, tokens, time_ms):
        self.stats["requests"] += 1
        self.stats["tokens_generated"] += tokens
        
        # Сохраняем в лог
        with open("qwen_logs.jsonl", "a") as f:
            log_entry = {
                "timestamp": datetime.now().isoformat(),
                "prompt_length": len(prompt),
                "response_length": len(response),
                "tokens": tokens,
                "response_time_ms": time_ms
            }
            f.write(json.dumps(log_entry) + "\n")
```

## 🎨 **Креативные применения**

### **1. Генерация контента:**
```python
def generate_content(topic, style="профессиональный"):
    prompt = f"""Напиши статью на тему '{topic}' в {style} стиле.
    
Включи:
- Заголовок
- Введение
- 3 основных раздела
- Заключение
- 5 ключевых выводов"""
    
    return ask_qwen(prompt)
```

### **2. Brainstorming помощник:**
```python
def brainstorm(topic, ideas_count=10):
    prompt = f"""Сгенерируй {ideas_count} идей на тему: {topic}
    
Формат:
1. Идея - Краткое описание
2. Идея - Краткое описание
..."""
    
    return ask_qwen(prompt)
```

### **3. Персональный тренер:**
```python
def create_learning_plan(topic, hours_per_day=2, days=30):
    prompt = f"""Создай план изучения '{topic}' на {days} дней
по {hours_per_day} часа в день.

Разбей по неделям, включай практические задания
и контрольные точки."""
    
    return ask_qwen(prompt)
```

## ⚡ **Производительность и оптимизация**

### **Параметры для скорости:**
```python
fast_params = {
    "num_predict": 256,      # Короткие ответы
    "temperature": 0.3,      # Меньше креативности
    "num_thread": 8,         # Больше потоков
    "num_batch": 1024,       # Больший батч
    "repeat_penalty": 1.0    # Минимальный штраф
}
```

### **Параметры для качества:**
```python
quality_params = {
    "num_predict": 1024,     # Длинные ответы
    "temperature": 0.8,      # Больше креативности
    "top_p": 0.95,           # Лучшее качество
    "top_k": 50,             # Больше разнообразия
    "mirostat": 2,           # Продвинутый контроль
    "repeat_penalty": 1.1    # Избегание повторов
}
```

## 🔄 **Работа в реальном времени**

### **Streaming с обработкой:**
```python
def stream_with_processing(prompt, callback):
    """Стриминг с обработкой каждого токена"""
    url = "http://localhost:11434/api/generate"
    
    with requests.post(url, json={
        "model": "qwen3:4b",
        "prompt": prompt,
        "stream": True
    }, stream=True) as r:
        
        full_response = ""
        for line in r.iter_lines():
            if line:
                data = json.loads(line)
                if "response" in data:
                    token = data["response"]
                    full_response += token
                    
                    # Колбэк для обработки каждого токена
                    if callback:
                        callback(token, full_response)
```

## 💰 **Экономия токенов**

### **Сжатие промптов:**
```python
def compress_prompt(prompt):
    """Сжимает промпт для экономии токенов"""
    compression_prompt = f"""Сожми этот текст, сохраняя смысл:
    
{prompt}

Сжатый текст:"""
    
    return ask_qwen(compression_prompt, num_predict=200)
```

## 🎯 **Итог: уникальные преимущества Qwen через Ollama**

1. **Полная приватность** - все локально
2. **Бесплатно** - нет лимитов на запросы
3. **Низкая задержка** - нет сетевых задержек
4. **Кастомизация** - можно дообучать и настраивать
5. **Оффлайн работа** - не требует интернета
6. **Интеграция** с любыми приложениями
7. **Контроль данных** - ваши данные не уходят к третьим лицам

**Для начала рекомендую:**
```python
# Простейший рабочий пример
import requests

def qwen_quick(prompt):
    response = requests.post(
        'http://localhost:11434/api/generate',
        json={
            'model': 'qwen3:4b',
            'prompt': prompt,
            'stream': False
        }
    )
    return response.json()['response']

# Тестируем все возможности
test_prompts = [
    "Напиши код сортировки пузырьком на Python",
    "Объясни теорию относительности простыми словами",
    "Переведи 'Hello world' на русский, французский и китайский",
    "Создай бизнес-план для стартапа",
    "Напиши поэму об искусственном интеллекте"
]

for prompt in test_prompts:
    print(f"\n❓ {prompt[:50]}...")
    print(f"🤖 {qwen_quick(prompt)[:100]}...")
```

Qwen через Ollama дает практически все те же возможности, что и облачные API (ChatGPT, Gemini), но с полным контролем и приватностью!








