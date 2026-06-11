## **Объявление массива в SEC(".maps")**
```c
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, u32);  // Индекс (обычно u32)
    __type(value, struct stats);
    __uint(max_entries, 1024);  // Фиксированный размер
} my_array SEC(".maps");
```

### **Ключевые особенности `BPF_MAP_TYPE_ARRAY`:**
- **Фиксированный размер** (не растёт динамически).
- **Ключ — это индекс (u32)**, обычно от `0` до `max_entries - 1`.
- **Значения хранятся в непрерывной памяти** (как обычный массив).
- **Операции доступа — O(1)** (прямая адресация по индексу).
- **Нельзя удалять элементы** (но можно перезаписывать).
- **Идеально для статических данных** (например, статистика, счётчики, конфигурация).

---

## **Основные операции (в eBPF-программе)**

### **1. Запись в массив**
```c
struct stats val = { .packets = 1, .bytes = 128 };
u32 index = 0;  // Индекс должен быть < max_entries
bpf_map_update_elem(&my_array, &index, &val, BPF_ANY);
```
- **`BPF_ANY`** — перезаписывает существующее значение.
- **`BPF_NOEXIST`** — выдаст ошибку, если элемент уже существует (бесполезно для массивов).
- **`BPF_EXIST`** — обновляет только существующий элемент.

### **2. Чтение из массива**
```c
u32 index = 0;
struct stats *pval = bpf_map_lookup_elem(&my_array, &index);
if (pval) {
    bpf_printk("Packets: %llu", pval->packets);
}
```
- **Возвращает указатель** на значение (или `NULL` при ошибке).

### **3. Итерация по массиву**
```c
u32 index = 0;
struct stats val;
for (; index < 1024; index++) {
    if (bpf_map_lookup_elem(&my_array, &index, &val) == 0) {
        bpf_printk("Index %d: %llu packets", index, val.packets);
    }
}
```
- Массивы **не поддерживают `bpf_map_get_next_key`** (в отличие от хешей).
- Итерация возможна только **линейным перебором**.

### **4. Удаление элементов**
```c
u32 index = 0;
bpf_map_delete_elem(&my_array, &index);  // НЕ РАБОТАЕТ!
```
- **`BPF_MAP_TYPE_ARRAY` не поддерживает удаление** (в отличие от `BPF_MAP_TYPE_HASH`).
- Можно только **обнулить значение**:
  ```c
  struct stats zero = {0};
  bpf_map_update_elem(&my_array, &index, &zero, BPF_ANY);
  ```

