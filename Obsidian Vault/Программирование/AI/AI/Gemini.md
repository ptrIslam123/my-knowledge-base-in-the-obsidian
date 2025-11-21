### **Вариант 2: Бесплатный облачный API (проще всего)**

**Google Gemini API - лучший старт:**

1. Получите API ключ: [Google AI Studio](https://aistudio.google.com/)
2. Установите библиотеку:
```bash
pip install google-generativeai
```

```python
import google.generativeai as genai

# Настройка
genai.configure(api_key="ВАШ_КЛЮЧ")  # бесплатно 60 запросов/мин

# Простое использование
model = genai.GenerativeModel('gemini-1.5-flash')
response = model.generate_content("Объясни квантовую физику просто")
print(response.text)

# Чат-бот
chat = model.start_chat(history=[])
response = chat.send_message("Привет!")
print(response.text)
```

## 🖥️ **Детальные варианты локального развертывания**

### **A. Для слабых машин (до 8ГБ RAM)**

**Используйте `transformers` + легкие модели:**

```bash
pip install transformers torch accelerate
```

```python
from transformers import pipeline, AutoTokenizer, AutoModelForCausalLM
import torch

# Выберите легкую модель
model_id = "microsoft/phi-2"  # 2.7B параметров
# или "TinyLlama/TinyLlama-1.1B-Chat-v1.0"  # 1.1B
# или "microsoft/Phi-3-mini-4k-instruct"  # 3.8B

# Загрузка модели (один раз, потом используйте из cache)
model = AutoModelForCausalLM.from_pretrained(
    model_id,
    torch_dtype=torch.float32,  # Используйте float16 если есть GPU
    device_map="auto"
)
tokenizer = AutoTokenizer.from_pretrained(model_id)

pipe = pipeline("text-generation", model=model, tokenizer=tokenizer)

# Генерация
result = pipe("Как настроить Python?")
print(result[0]['generated_text'])
```

### **B. Для средних машин (8-16ГБ RAM)**

**Используйте Ollama с оптимизацией:**

```bash
# Установите Ollama
# Скачайте оптимизированную модель
ollama pull llama3.2:1b  # 1B параметров
ollama pull mistral:7b  # 7B (4-bit квантованная)
```

```python
import requests
import json

class LocalAIClient:
    def __init__(self, model="mistral:7b", base_url="http://localhost:11434"):
        self.base_url = base_url
        self.model = model
        
    def generate(self, prompt, max_tokens=500):
        response = requests.post(
            f'{self.base_url}/api/generate',
            json={
                'model': self.model,
                'prompt': prompt,
                'options': {
                    'num_predict': max_tokens,
                    'temperature': 0.7
                },
                'stream': False
            },
            timeout=300
        )
        return response.json()['response']
    
    def chat(self, messages):
        """Формат как у OpenAI"""
        response = requests.post(
            f'{self.base_url}/api/chat',
            json={
                'model': self.model,
                'messages': messages,
                'stream': False
            }
        )
        return response.json()['message']['content']

# Использование
ai = LocalAIClient(model="llama3.2:1b")
response = ai.generate("Напиши код сортировки на Python")
print(response)
```

## ☁️ **Облачные REST API (бесплатные)**

### **1. Groq API (самый быстрый)**

```python
# pip install groq
from groq import Groq

client = Groq(api_key="ВАШ_КЛЮЧ")  # Бесплатно, регистрация на groq.com

completion = client.chat.completions.create(
    model="llama3-70b-8192",  # или "mixtral-8x7b-32768"
    messages=[
        {"role": "user", "content": "Объясни ООП"}
    ],
    temperature=0.7,
    max_tokens=1024
)
print(completion.choices[0].message.content)
```

### **2. OpenRouter (много моделей)**

```python
import requests

headers = {
    "Authorization": "Bearer ВАШ_КЛЮЧ",  # Бесплатные модели есть
    "Content-Type": "application/json"
}

data = {
    "model": "google/gemini-flash-1.5",  # Или "meta-llama/llama-3-8b-instruct"
    "messages": [{"role": "user", "content": "Привет"}]
}

response = requests.post(
    "https://openrouter.ai/api/v1/chat/completions",
    headers=headers,
    json=data
)
print(response.json()['choices'][0]['message']['content'])
```

### **3. Встроенный в VSCode (очень просто)**

Если используете VSCode:
1. Установите расширение "Continue"
2. Нажмите Ctrl+Shift+P → "Continue: Use Local Model"
3. Выберите модель
4. Используйте из Python через их API

## 🚀 **Готовый шаблон для начала**

**Файл `ai_helper.py`:**
```python
import os
from enum import Enum
from typing import Optional

class AIMode(Enum):
    LOCAL = "local"
    GEMINI = "gemini"
    GROQ = "groq"

class AIAssistant:
    def __init__(self, mode: AIMode = AIMode.GEMINI):
        self.mode = mode
        self.setup_client()
    
    def setup_client(self):
        if self.mode == AIMode.LOCAL:
            # Локальный Ollama
            self.base_url = "http://localhost:11434"
            self.model = "phi3"
        elif self.mode == AIMode.GEMINI:
            import google.generativeai as genai
            genai.configure(api_key=os.getenv("GEMINI_API_KEY"))
            self.model = genai.GenerativeModel('gemini-1.5-flash')
        elif self.mode == AIMode.GROQ:
            from groq import Groq
            self.client = Groq(api_key=os.getenv("GROQ_API_KEY"))
    
    def ask(self, question: str) -> str:
        if self.mode == AIMode.LOCAL:
            import requests
            response = requests.post(
                f'{self.base_url}/api/generate',
                json={'model': self.model, 'prompt': question, 'stream': False}
            )
            return response.json()['response']
        
        elif self.mode == AIMode.GEMINI:
            response = self.model.generate_content(question)
            return response.text
        
        elif self.mode == AIMode.GROQ:
            completion = self.client.chat.completions.create(
                model="llama3-70b-8192",
                messages=[{"role": "user", "content": question}]
            )
            return completion.choices[0].message.content

# Использование
if __name__ == "__main__":
    # 1. Для облака (самый простой)
    # assistant = AIAssistant(AIMode.GEMINI)
    
    # 2. Для локального
    assistant = AIAssistant(AIMode.LOCAL)
    
    response = assistant.ask("Напиши функцию для чтения файла на Python")
    print(response)
```

## 📊 **Сравнение вариантов**

| Вариант | Сложность | Скорость | Качество | Затраты |
|---------|-----------|----------|----------|---------|
| **Gemini API** | ★☆☆☆☆ | ★★★★★ | ★★★★★ | Бесплатно |
| **Ollama локально** | ★★☆☆☆ | ★★☆☆☆ | ★★★☆☆ | Бесплатно |
| **Groq Cloud** | ★☆☆☆☆ | ★★★★★ | ★★★★★ | Бесплатно |
| **Transformers локально** | ★★★☆☆ | ★☆☆☆☆ | ★★☆☆☆ | Бесплатно |

## 🎯 **Мой совет по порядку действий:**

1. **Начните с Gemini API** - получите ключ, попробуйте мой код выше
2. **Если нужна локальная работа** - установите Ollama + phi3
3. **Для программирования** - добавьте Groq API для сложных задач
4. **Для продакшена** - используйте кэширование и асинхронные запросы

**Минимальный рабочий пример:**
```bash
# 1. Установите зависимости
pip install requests google-generativeai

# 2. Получите бесплатный ключ на https://aistudio.google.com/

# 3. Запустите этот код
```

```python
# simple_ai.py
import os
os.environ["GEMINI_API_KEY"] = "ваш_ключ"

import google.generativeai as genai
genai.configure(api_key=os.environ["GEMINI_API_KEY"])

model = genai.GenerativeModel('gemini-1.5-flash')
response = model.generate_content(input("Ваш вопрос: "))
print("\nОтвет:", response.text)
```

Какой у вас тип задач? Кодирование, анализ текста, чат-бот? И какие характеристики у вашей машины (RAM, CPU, GPU)?