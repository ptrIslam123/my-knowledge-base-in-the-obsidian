Эти структуры данных предоставляют **FIFO (очередь)** и **LIFO (стек)** семантику внутри eBPF-программ. Они работают **без ключей** — элементы добавляются и извлекаются по порядку.

---

## **1. `BPF_MAP_TYPE_QUEUE` (очередь, FIFO)**
### **Объявление**
```c
struct {
    __uint(type, BPF_MAP_TYPE_QUEUE);
    __type(value, struct event);  // Тип элемента
    __uint(max_entries, 1024);   // Максимальная длина очереди
} event_queue SEC(".maps");
```
- **Нет ключа (`key`)** — только значение (`value`).
- **Работает по принципу FIFO** (First In, First Out).
- **Пример использования**: передача событий из ядра в пользовательское пространство.

### **Основные операции**
#### **1. `bpf_map_push_elem()` — добавление в конец очереди**
```c
struct event e = { .pid = 123, .action = 1 };
bpf_map_push_elem(&event_queue, &e, BPF_ANY);
```
- `BPF_ANY` — единственный поддерживаемый флаг.
- **При переполнении (`max_entries` достигнут) возвращает ошибку**.

#### **2. `bpf_map_pop_elem()` — извлечение из начала очереди**
```c
struct event e;
if (bpf_map_pop_elem(&event_queue, &e) == 0) {
    bpf_printk("Event: pid=%d", e.pid);
}
```
- **Если очередь пуста — возвращает ошибку**.

---

## **2. `BPF_MAP_TYPE_STACK` (стек, LIFO)**
### **Объявление**
```c
struct {
    __uint(type, BPF_MAP_TYPE_STACK);
    __type(value, struct event);  // Тип элемента
    __uint(max_entries, 1024);   // Максимальная глубина стека
} event_stack SEC(".maps");
```
- **Нет ключа (`key`)** — только значение (`value`).
- **Работает по принципу LIFO** (Last In, First Out).
- **Пример использования**: обработка событий в обратном порядке.

### **Основные операции**
#### **1. `bpf_map_push_elem()` — добавление в стек**
```c
struct event e = { .pid = 123, .action = 1 };
bpf_map_push_elem(&event_stack, &e, BPF_ANY);
```
- Аналогично очереди, но извлекаться будет **последний добавленный элемент**.

#### **2. `bpf_map_pop_elem()` — извлечение из стека**
```c
struct event e;
if (bpf_map_pop_elem(&event_stack, &e) == 0) {
    bpf_printk("Event: pid=%d", e.pid);
}
```
- **Возвращает самый новый элемент** (последний добавленный).

---

## **3. Сравнение QUEUE и STACK**
| Характеристика       | `BPF_MAP_TYPE_QUEUE` (FIFO)       | `BPF_MAP_TYPE_STACK` (LIFO)       |
|-----------------------|-----------------------------------|-----------------------------------|
| **Семантика**         | Первый вошёл — первый вышел       | Последний вошёл — первый вышел    |
| **Использование**     | - Обработка событий по порядку    | - Отмена операций (как `Ctrl+Z`)  |
| **Пример из жизни**   | Очередь задач                     | Стек вызовов / история изменений  |

---

## **4. Особенности и ограничения**
1. **Нет поддержки ключей** — только значения.
2. **Фиксированный размер** (`max_entries`).
3. **Не поддерживают**:
   - `bpf_map_lookup_elem` (поиск по ключу).
   - `bpf_map_delete_elem` (удаление по ключу).
   - `bpf_map_get_next_key` (итерация).
4. **Доступны только**:
   - `bpf_map_push_elem` (добавление).
   - `bpf_map_pop_elem` (извлечение).

---

## **5. Примеры использования**
### **1. Очередь событий (FIFO)**
```c
// eBPF-программа (ядро)
struct event {
    u32 pid;
    u8 action;
};

struct {
    __uint(type, BPF_MAP_TYPE_QUEUE);
    __type(value, struct event);
    __uint(max_entries, 100);
} events SEC(".maps");

// При событии добавляем в очередь
struct event e = { .pid = bpf_get_current_pid_tgid(), .action = 1 };
bpf_map_push_elem(&events, &e, BPF_ANY);
```

**Пользовательское пространство (Python + libbpf):**
```python
while True:
    try:
        e = bpf["events"].pop()  # Чтение из очереди
        print(f"PID: {e.pid}, Action: {e.action}")
    except Exception:
        break
```

### **2. Стек для отката изменений (LIFO)**
```c
// eBPF-программа (ядро)
struct config_change {
    u32 old_value;
    u32 new_value;
};

struct {
    __uint(type, BPF_MAP_TYPE_STACK);
    __type(value, struct config_change);
    __uint(max_entries, 10);
} config_stack SEC(".maps");

// При изменении конфига сохраняем старое значение
struct config_change ch = { .old_value = 100, .new_value = 200 };
bpf_map_push_elem(&config_stack, &ch, BPF_ANY);

// Откат последнего изменения
struct config_change last;
if (bpf_map_pop_elem(&config_stack, &last) == 0) {
    apply_config(last.old_value);  // Восстанавливаем старую конфигурацию
}
```

---

## **6. Производительность**
- **Операции `push`/`pop` работают за O(1)**.
- **Нет блокировок** (в `PERCPU`-режиме возможна параллельная работа).
