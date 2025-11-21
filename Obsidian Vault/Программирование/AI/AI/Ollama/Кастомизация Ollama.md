## **Уровни специализации моделей**

### **1. Быстрая специализация через промпт-инжиниринг (Самый простой)**

```python
class SpecialistModel:
    def __init__(self, model="qwen3:4b"):
        self.model = model
        self.system_prompts = {
            "coder": """Ты - экспертный программист-ассистент. 
Твои правила:
1. Отвечай только кодом с минимальными объяснениями
2. Комментируй сложные части кода
3. Предлагай оптимизации
4. Учитывай best practices языка
5. Не обсуждай не-программистские темы""",
            
            "translator": """Ты - профессиональный переводчик.
Правила:
6. Сохраняй стиль и тон оригинала
7. Учитывай культурный контекст
8. Для технических терминов давай примечания
9. Предлагай несколько вариантов перевода
10. Исправляй грамматические ошибки в оригинале""",
            
            "finance": """Ты - финансовый консультант.
Правила:
11. Давай точные расчеты с формулами
12. Учитывай налоговые последствия
13. Предупреждай о рисках
14. Не давай конкретных инвестиционных советов
15. Ссылайся на актуальные нормативные акты""",
            
            "lawyer": """Ты - юридический ассистент.
Правила:
16. Цитируй статьи законов
17. Указывай юрисдикцию
18. Различай рекомендации и требования закона
19. Предупреждай о необходимости консультации реального юриста
20. Объясняй юридические термины простым языком""",
            
            "teacher": """Ты - преподаватель.
Правила:
21. Адаптируй объяснение под уровень ученика
22. Давай примеры и аналогии
23. Проверяй понимание вопросами
24. Предоставляй дополнительные ресурсы
25. Разбивай сложные темы на шаги"""
        }
    
    def ask(self, prompt, specialty="coder", temperature=0.1):
        """Задать вопрос специализированной модели"""
        system_msg = self.system_prompts.get(specialty, "")
        
        full_prompt = f"""{system_msg}

Запрос пользователя: {prompt}

Ответ специалиста:"""
        
        return self._call_model(full_prompt, temperature)
    
    def _call_model(self, prompt, temperature):
        import requests
        response = requests.post(
            'http://localhost:11434/api/generate',
            json={
                'model': self.model,
                'prompt': prompt,
                'stream': False,
                'options': {'temperature': temperature}
            }
        )
        return response.json()['response']

# Использование
specialist = SpecialistModel()
code = specialist.ask("Напиши REST API на FastAPI", specialty="coder")
translation = specialist.ask("Translate technical documentation", specialty="translator")
```

### **2. Fine-tuning (дообучение на своих данных)**

#### **A. Создание датасета для специализации**
```python
# Пример датасета для кодогенерации
coding_dataset = [
    {
        "instruction": "Напиши функцию для проверки числа на простоту",
        "input": "",
        "output": "def is_prime(n):\n    if n < 2:\n        return False\n    for i in range(2, int(n**0.5) + 1):\n        if n % i == 0:\n            return False\n    return True"
    },
    {
        "instruction": "Создай класс для работы с базой данных",
        "input": "SQLite, CRUD операции",
        "output": "import sqlite3\n\nclass Database:\n    def __init__(self, db_path):\n        self.conn = sqlite3.connect(db_path)\n        self.cursor = self.conn.cursor()\n    \n    def create_table(self, table_name, columns):\n        query = f\"CREATE TABLE IF NOT EXISTS {table_name} ({columns})\"\n        self.cursor.execute(query)\n        self.conn.commit()\n    \n    # ... остальные методы"
    }
]

# Дамп в JSONL для обучения
import json
with open('coding_dataset.jsonl', 'w') as f:
    for item in coding_dataset:
        f.write(json.dumps(item, ensure_ascii=False) + '\n')
```

#### **B. Обучение с помощью Ollama (Modelfile)**
```dockerfile
# coding_specialist.Modelfile
FROM qwen3:4b

# Системный промпт
SYSTEM """Ты - эксперт по программированию.
Ты специализируешься на генерации чистого, эффективного кода.
Ты понимаешь лучшие практики и шаблоны проектирования.
Ты объясняешь код только когда явно просят."""

# Параметры
PARAMETER temperature 0.1
PARAMETER top_p 0.9
PARAMETER num_ctx 8192

# Примеры для few-shot обучения
TEMPLATE """{{ .System }}

Пример 1:
Вопрос: Напиши функцию для обратного массива
Ответ: 
```python
def reverse_array(arr):
    return arr[::-1]
```

Пример 2:
Вопрос: Создай синглтон класс на Python
Ответ:
```python
class Singleton:
    _instance = None
    
    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance
```

Теперь ответь на вопрос: {{ .Prompt }}"""
```

**Создание специализированной модели:**
```bash
# Создаем Modelfile
cat > coding_specialist.Modelfile << 'EOF'
FROM qwen3:4b
SYSTEM "Ты - эксперт по программированию на Python, JavaScript и SQL."
PARAMETER temperature 0.1
EOF

# Создаем модель
ollama create coding-specialist -f coding_specialist.Modelfile

# Тестируем
ollama run coding-specialist "Напиши функцию для бинарного поиска"
```

### **3. LoRA адаптеры (легкое дообучение)**
```python
# Использование библиотеки для LoRA
# pip install peft transformers torch

from peft import LoraConfig, get_peft_model, TaskType
from transformers import AutoModelForCausalLM, AutoTokenizer
import torch

# Загружаем базовую модель
model_name = "Qwen/Qwen2.5-4B-Instruct"
model = AutoModelForCausalLM.from_pretrained(
    model_name,
    torch_dtype=torch.float16,
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(model_name)

# Конфигурация LoRA для кодогенерации
lora_config = LoraConfig(
    task_type=TaskType.CAUSAL_LM,
    r=16,  # Rank
    lora_alpha=32,
    target_modules=["q_proj", "v_proj", "k_proj", "o_proj", "gate_proj", "up_proj", "down_proj"],
    lora_dropout=0.1,
    bias="none"
)

# Применяем LoRA
model = get_peft_model(model, lora_config)

# Тренируем на своих данных...
# Затем сохраняем адаптер
model.save_pretrained("./coding_lora_adapter")
```

### **4. RAG (Retrieval Augmented Generation) - специализация через документы**

```python
import chromadb
from sentence_transformers import SentenceTransformer

class RAGSpecialist:
    def __init__(self, specialty_docs, model="qwen3:4b"):
        self.model = model
        self.embedder = SentenceTransformer('all-MiniLM-L6-v2')
        self.chroma_client = chromadb.Client()
        
        # Создаем коллекцию для специализации
        self.collection = self.chroma_client.create_collection(
            name=f"{specialty}_knowledge"
        )
        
        # Индексируем документы
        self._index_documents(specialty_docs)
    
    def _index_documents(self, docs):
        """Индексируем документы по специальности"""
        for i, doc in enumerate(docs):
            embedding = self.embedder.encode(doc).tolist()
            self.collection.add(
                embeddings=[embedding],
                documents=[doc],
                ids=[str(i)]
            )
    
    def query(self, question, k=3):
        """Поиск релевантной информации и генерация ответа"""
        # Поиск релевантных документов
        query_embedding = self.embedder.encode(question).tolist()
        results = self.collection.query(
            query_embeddings=[query_embedding],
            n_results=k
        )
        
        # Собираем контекст
        context = "\n".join(results['documents'][0])
        
        # Создаем промпт с контекстом
        prompt = f"""Ты - специалист. Используй информацию ниже:

Контекст:
{context}

Вопрос: {question}

Ответ специалиста:"""
        
        return self._call_model(prompt)
    
    def _call_model(self, prompt):
        import requests
        response = requests.post(
            'http://localhost:11434/api/generate',
            json={'model': self.model, 'prompt': prompt, 'stream': False}
        )
        return response.json()['response']

# Использование для финансового помощника
finance_docs = [
    "Налог на доходы физических лиц: 13% для резидентов",
    "ИИС типа А: вычет 13% от взносов до 400к в год",
    "Дивиденды облагаются по ставке 13%",
    "Ключевая ставка ЦБ: 16% с декабря 2023"
]

finance_expert = RAGSpecialist(finance_docs, specialty="finance")
answer = finance_expert.query("Какие налоги с дивидендов?")
```

### **5. Мультимодельный подход (экспертная система)**

```python
class ExpertSystem:
    def __init__(self):
        self.experts = {
            "coding": {
                "model": "codellama:7b",
                "temperature": 0.1,
                "prompt": "Ты CodeLlama - специалист по программированию."
            },
            "translation": {
                "model": "qwen:1.8b",
                "temperature": 0.3,
                "prompt": "Ты профессиональный переводчик с поддержкой 100+ языков."
            },
            "finance": {
                "model": "mistral:7b",
                "temperature": 0.2,
                "prompt": "Ты финансовый аналитик с MBA."
            },
            "creative": {
                "model": "llama3.2:3b",
                "temperature": 0.8,
                "prompt": "Ты креативный писатель и поэт."
            }
        }
    
    def route_to_expert(self, query):
        """Определяет, какой эксперт нужен"""
        query_lower = query.lower()
        
        if any(word in query_lower for word in ['код', 'программ', 'функц', 'алгоритм']):
            return "coding"
        elif any(word in query_lower for word in ['перевод', 'translate', 'language']):
            return "translation"
        elif any(word in query_lower for word in ['финанс', 'налог', 'инвест', 'деньги']):
            return "finance"
        else:
            return "creative"
    
    def ask(self, query):
        expert_type = self.route_to_expert(query)
        expert = self.experts[expert_type]
        
        import requests
        response = requests.post(
            'http://localhost:11434/api/generate',
            json={
                'model': expert["model"],
                'prompt': f"{expert['prompt']}\n\nВопрос: {query}",
                'options': {'temperature': expert['temperature']},
                'stream': False
            }
        )
        
        return {
            'expert': expert_type,
            'answer': response.json()['response']
        }

# Использование
system = ExpertSystem()
result = system.ask("Напиши алгоритм быстрой сортировки")
print(f"Эксперт: {result['expert']}")
print(f"Ответ: {result['answer']}")
```

### **6. Специализация через quantization (оптимизация под задачу)**

```bash
# Создание специализированной квантованной модели
# 4-bit quantization для экономии памяти

# Для кодогенерации (нужна точность)
ollama pull codellama:7b-q4_0

# Для чата (можно более агрессивную квантзацию)
ollama pull qwen:1.8b-q2_K  # 2-bit quantization

# Создаем свой пресет
cat > finance-expert.Modelfile << EOF
FROM mistral:7b-q4_0
SYSTEM "Ты финансовый эксперт. Говори только о финансах."
PARAMETER temperature 0.2
PARAMETER num_ctx 4096
TEMPLATE "Financial Expert: {{ .Prompt }}\nAI Financial Analyst:"
EOF

ollama create finance-expert -f finance-expert.Modelfile
```

### **7. Практические примеры специализации**

#### **Чистый переводчик:**
```python
class ProfessionalTranslator:
    def __init__(self):
        self.languages = {
            'en': 'английский',
            'ru': 'русский', 
            'zh': 'китайский',
            'es': 'испанский',
            'fr': 'французский',
            'de': 'немецкий'
        }
    
    def translate(self, text, source='auto', target='ru', style='formal'):
        prompt = f"""Ты профессиональный переводчик.
            
Исходный текст: "{text}"
            
Переведи с {self.languages.get(source, source)} на {self.languages[target]}.
Стиль перевода: {style}.
            
Перевод:"""
        
        # Используем очень низкую температуру для точности
        return self._call_model(prompt, temperature=0.01)
```

#### **Программист-ассистент:**
```python
class CodeAssistant:
    def __init__(self, language="python"):
        self.language = language
        self.templates = {
            'python': {
                'docstring': '"""{}"""',
                'type_hints': True,
                'style': 'pep8'
            },
            'javascript': {
                'docstring': '/**\n * {} \n */',
                'type_hints': False,
                'style': 'airbnb'
            }
        }
    
    def generate_code(self, description, include_tests=True):
        prompt = f"""Generate {self.language} code.
        
Requirements: {description}
        
Include:
1. Complete implementation
2. Comments for complex parts
3. Error handling
4. {'Unit tests' if include_tests else 'No tests needed'}
        
Follow {self.templates[self.language]['style']} style.
        
Code:"""
        
        return self._call_model(prompt, temperature=0.1)
```

#### **Финансовый консультант:**
```python
class FinancialAdvisor:
    def __init__(self, country="RU"):
        self.country = country
        self.tax_rules = self._load_tax_rules(country)
    
    def calculate_investment(self, principal, rate, years, tax_included=True):
        prompt = f"""Financial calculation with explanation.
        
Principal: {principal} {self.currency}
Annual rate: {rate}%
Years: {years}
Tax included: {tax_included}
Country: {self.country}
        
Calculate:
1. Final amount
2. Total profit
3. Monthly payments if applicable
4. Tax implications
        
Use formulas and show calculations step by step."""
        
        return self._call_model(prompt, temperature=0.1)
```

### **8. Создание полностью специализированного сервиса**

```python
# specialized_api.py
from fastapi import FastAPI
from pydantic import BaseModel
import requests

app = FastAPI()

class Query(BaseModel):
    text: str
    specialty: str = "general"

class Specialist:
    def __init__(self):
        self.configs = {
            "coder": {
                "model": "codellama:7b",
                "system": "Ты CodeLlama - эксперт по программированию.",
                "temp": 0.1
            },
            "translator": {
                "model": "qwen:4b",
                "system": "Ты переводчик с 15+ лет опыта.",
                "temp": 0.3
            },
            "finance": {
                "model": "mistral:7b", 
                "system": "Ты сертифицированный финансовый аналитик (CFA).",
                "temp": 0.2
            }
        }
    
    def query(self, text: str, specialty: str) -> str:
        config = self.configs.get(specialty, self.configs["general"])
        
        response = requests.post(
            'http://localhost:11434/api/generate',
            json={
                'model': config['model'],
                'prompt': f"{config['system']}\n\n{text}",
                'options': {'temperature': config['temp']},
                'stream': False
            }
        )
        return response.json()['response']

specialist = Specialist()

@app.post("/api/specialist")
async def ask_specialist(query: Query):
    return {"answer": specialist.query(query.text, query.specialty)}

# Запуск: uvicorn specialized_api:app --reload
```

## 📊 **Сравнение методов специализации**

| Метод | Сложность | Эффективность | Время | Память |
|-------|-----------|---------------|-------|--------|
| **Промпт-инжиниринг** | ★☆☆☆☆ | ★★☆☆☆ | Мгновенно | 0 MB |
| **System Prompt** | ★☆☆☆☆ | ★★★☆☆ | Мгновенно | 0 MB |
| **Modelfile** | ★★☆☆☆ | ★★★★☆ | 1 мин | +100MB |
| **LoRA адаптеры** | ★★★★☆ | ★★★★★ | 1-4 часа | +50MB |
| **Full Fine-tune** | ★★★★★ | ★★★★★ | 4-24 часа | +4GB |
| **RAG** | ★★★☆☆ | ★★★★☆ | 5 мин | Зависит от данных |

## 🚀 **Рекомендации по специализации**

### **Начните с этого:**
```bash
# 1. Создайте Modelfile для вашей специализации
cat > my_specialist.Modelfile << EOF
FROM qwen3:4b
SYSTEM "Ты специализируешься на [ВАША_ТЕМА]. Отвечай только в контексте этой темы."
PARAMETER temperature 0.1
PARAMETER num_predict 1000
EOF

# 2. Создайте модель
ollama create my-specialist -f my_specialist.Modelfile

# 3. Протестируйте
ollama run my-specialist "Ваш вопрос по специализации"
```

### **Готовые шаблоны для разных специализаций:**

```python
SPECIALTY_TEMPLATES = {
    "coder": {
        "system": """Ты - Senior разработчик с 10+ лет опыта.
Правила:
1. Пиши чистый, production-ready код
2. Добавляй комментарии только к сложной логике
3. Учитывай edge cases
4. Предлагай альтернативные решения
5. Следуй best practices языка""",
        "temperature": 0.1,
        "model": "qwen3:4b"
    },
    
    "translator": {
        "system": """Ты - профессиональный переводчик.
Правила:
6. Сохраняй терминологию
7. Адаптируй идиомы
8. Указывай неоднозначности
9. Предлагай варианты перевода
10. Сохраняй регистр и форматирование""",
        "temperature": 0.3,
        "model": "qwen3:4b"
    },
    
    "finance": {
        "system": """Ты - финансовый консультант.
Правила:
11. Проверяй актуальность данных
12. Указывай источники информации
13. Различай факты и рекомендации
14. Рассчитывай с формулами
15. Учитывай налоговые аспекты""",
        "temperature": 0.2,
        "model": "mistral:7b"
    }
}
```

## ✅ **Итог:**
**Да, вы можете специализировать Qwen и другие модели под любую задачу!** Самый быстрый способ — через system prompt и Modelfile. Для глубокой специализации — LoRA или fine-tuning. Для каждой задачи можно создать отдельную "экспертную" модель, оптимизированную именно под нее.