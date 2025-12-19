### **Вариант 1: Простой локальный AI (5 минут установки)**
# Установите Ollama
```bash

curl -fsSL https://ollama.com/install.sh | sh
```

# Скачивание модели
```bash
ollama pull [model]
```

| модели       | Правильно   | Примечание                                 |
| ------------ | ----------- | ------------------------------------------ |
| `qwen3:0.5b` | `qwen:0.5b` | `qwen` = Qwen2/Qwen3 в зависимости от тега |
| `qwen3:1.8b` | `qwen:1.8b` | **ЭТО ТО, ЧТО ТЕБЕ НУЖНО**                 |
| `qwen3:4b`   | `qwen:4b`   | Или `qwen2:4b` — тоже работает             |
| `qwen3:8b`   | `qwen:8b`   | Тяжело на CPU                              |
| `qwen3:32b`  | `qwen2:72b` | Не для слабых ПК                           |

**Python код для работы:**
```python
import requests
import json

def stream_qwen(prompt: str):
	url = "http://localhost:11434/api/generate"
	payload = {
		"model": "qwen3:4b",
		"prompt": prompt,
		"stream": False,
	}
	
	with requests.post(url, json=payload, stream=True) as r:
		for line in r.iter_lines():
			if line:
				data = json.loads(line)
				if "response" in data:
					print(data["response"], end="", flush=True)
				if data.get("done"):
					print("\n✅ Done.")
					break

# Запуск:
stream_qwen("Hello!")
```


