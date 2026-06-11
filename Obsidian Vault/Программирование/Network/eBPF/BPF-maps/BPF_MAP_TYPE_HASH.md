## **Определение хеш таблиц в eBPF**

#### **Объявление в SEC(".maps")**
```c
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, u32);
    __type(value, struct stats);
    __uint(max_entries, 1024);
} my_map SEC(".maps");
```

## **`struct { ... } my_map`**

- Создаёт анонимную структуру с именем `my_map`.
- В eBPF это **объект карты**, который будет доступен как в программе ядра, так и в пользовательском пространстве.

## **`__uint(type, BPF_MAP_TYPE_HASH)`**

- **`__uint`**: Макрос для объявления unsigned int-поля в BTF (BPF Type Format)
- **`type`**: Указывает тип карты, в данном случае 
- **`BPF_MAP_TYPE_HASH`**:
    - Тип хеш-таблицы (ключ → значение).
    - Аналогичен `std::unordered_map` в C++.
    - **Особенности**:
        - Динамически расширяется (до `max_entries`).
        - Ключи не упорядочены.
        - Сложность операций: O(1).

## **`__type(key, u32)`**

- **`__type`**: Макрос для указания типа поля с учётом BTF.
- **`key`**: Тип ключа карты.
- **`u32`**: 32-битный беззнаковый integer (uint32_t).

**Особенности ключей**
- Должны быть **фиксированного размера**.
- Могут быть сложными структурами (но не рекомендуется для производительности).
- Пример сложного ключа:
```c
struct my_key {
    u32 ip;
    u16 port;
};
__type(key, struct my_key);
```

## **`__type(value, struct stats)`**

- **`value`**: Тип значения карты.
- **`struct stats`**: Пользовательская структура (может быть любой).

**Пример определения `struct stats`**:
```c
struct stats {
    u64 packets;
    u64 bytes;
    u32 last_update;
};
```

**Ограничения значений**:
- Размер значения должен быть **фиксированным**.
- Максимальный размер — обычно 8KB (зависит от ядра).

## **`__uint(max_entries, 1024)`**

- **`max_entries`**: Максимальное количество элементов в карте.
- **`1024`**: Конкретное значение (может быть до 1М+ для некоторых типов).

**Важно**:
- Для `BPF_MAP_TYPE_HASH` — это **примерный** лимит (реальное число может быть больше).
- Для `BPF_MAP_TYPE_ARRAY` — это **точный** размер (массив не растёт динамически).

## **`SEC(".maps")`**

- **`SEC`**: Макрос для указания секции ELF-файла.
- **`.maps`**: Специальная секция, где libbpf ищет определения карт.

**Что происходит на этапе загрузки**:
1. Компилятор (clang) помещает карту в секцию `.maps`.
2. Загрузчик (libbpf) создаёт карту в ядре при загрузке программы.
3. Возвращает файловый дескриптор (`map_fd`) для доступа из пользовательского пространства.
#### **Основные операции с хеш таблицами (Kernel space)**
```c
/* В eBPF-программе (ядро) */
// Вставка
struct stats val = {.count = 1};
bpf_map_update_elem(&my_map, &key, &val, BPF_ANY);

// Поиск
struct stats *pval = bpf_map_lookup_elem(&my_map, &key);

// Удаление
bpf_map_delete_elem(&my_map, &key);

// Итерация
u32 next_key;
bpf_map_get_next_key(&my_map, &key, &next_key);
```


## **4. Пример использования**
### **Статистика по протоколам (L4)**
```c
struct proto_stats {
    u64 packets;
    u64 bytes;
};

struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, u32);  // Индекс = номер протокола (IPPROTO_TCP=6, IPPROTO_UDP=17)
    __type(value, struct proto_stats);
    __uint(max_entries, 256);  // Достаточно для всех номеров протоколов
} stats_map SEC(".maps");

// В XDP-программе:
u32 proto = ctx->protocol;  // Например, IPPROTO_TCP=6
struct proto_stats *stats = bpf_map_lookup_elem(&stats_map, &proto);
if (stats) {
    stats->packets++;
    stats->bytes += ctx->data_end - ctx->data;
}
```

---

## **5. Ограничения**
1. **Нельзя изменять `max_entries`** после создания.
2. **Ключ всегда `u32`** (индекс).
3. **Нет автоматического вытеснения** (в отличие от `BPF_MAP_TYPE_LRU_HASH`).
4. **Нет поддержки `per-CPU` варианта** (для этого есть `BPF_MAP_TYPE_PERCPU_ARRAY`).

---

## **6. Вариации массива**
- **`BPF_MAP_TYPE_PERCPU_ARRAY`** — отдельная копия массива для каждого CPU (полезно для статистики без блокировок).
- **`BPF_MAP_TYPE_PROG_ARRAY`** — массив eBPF-программ (для динамического вызова).

---

### **Итог**
- **`BPF_MAP_TYPE_ARRAY`** — это быстрый статический массив с фиксированным размером.
- **Используется** для:
  - Счётчиков (например, статистика по протоколам).
  - Конфигурационных параметров (например, порты или IP-адреса).
  - Любых данных, где ключ — это индекс (`u32`).

## **4. Пользовательский доступ**

#### **Через syscall**
```c
// Чтение
union bpf_attr attr = {
    .map_fd = map_fd,
    .key = ptr_to_key,
    .value = ptr_to_value,
};
syscall(__NR_bpf, BPF_MAP_LOOKUP_ELEM, &attr, sizeof(attr));
```

#### **Через libbpf (User space)**
```c
#include <bpf/bpf.h>

int map_fd = bpf_obj_get("/sys/fs/bpf/my_map");
struct stats val;
u32 key = 42;

// Чтение
bpf_map_lookup_elem(map_fd, &key, &val);

// Запись
val.bytes += 512;
bpf_map_update_elem(map_fd, &key, &val, BPF_EXIST);
```


## **Подводные камни**

1. **Размер ключа/значения**:
    - Должны точно совпадать с объявлением.
    - Ошибка приведёт к `-EINVAL` при загрузке.
2. **BTF-информация**:
    - `__type` и `__uint` требуют поддержки BTF (ядра ≥5.2).
3. **Производительность**:
    - `BPF_MAP_TYPE_HASH` медленнее `BPF_MAP_TYPE_ARRAY` для последовательных ключей.


## **Отладка**

Проверить созданную карту:
```bash
# Поиск ID карты
bpftool map list

# Дамп содержимого
bpftool map dump id <MAP_ID>
```

---




## **5. Атомарные операции**

```c
// Атомарное сложение
__sync_fetch_and_add(&val->counter, 1);

// CAS (Compare-And-Swap)
bpf_spin_lock(&lock);
val->data = new_value;
bpf_spin_unlock(&lock);
```

---

## **6. Производительность и ограничения**


#### **Ключевые ограничения**
- Максимальный размер значения: 8KB (в большинстве реализаций)
- Глубина вложенности tail calls: 32
- Максимальное количество элементов: 1M (зависит от типа)

---

## **7. Практические примеры**

#### **Счётчик пакетов по IP**
```c
struct {
    __uint(type, BPF_MAP_TYPE_LRU_HASH);
    __type(key, __be32); // IP-адрес
    __type(value, u64);  // Счётчик
    __uint(max_entries, 100000);
} pkt_counts SEC(".maps");
```

#### **Передача событий в userspace**
```c
struct {
    __uint(type, BPF_MAP_TYPE_RINGBUF);
    __uint(max_entries, 1 << 20);
} events SEC(".maps");

struct event {
    u32 pid;
    char comm[16];
};
bpf_ringbuf_output(&events, &e, sizeof(e), 0);
```

---

# Полезные ресурсы
https://docs.ebpf.io/linux/helper-function/

### **Заключение**
eBPF Maps — это мощный механизм для:
- Разделяемого состояния между программами
- Эффективной передачи данных ядро↔пользовательское пространство
- Реализации сложных структур данных на уровне ядра

Для максимальной производительности важно:
1. Правильно выбирать тип карты под задачу
2. Минимизировать contention (использовать PERCPU)
3. Использовать batch-операции при массовых обработках

Примеры готовых решений:
- [Cilium's map management](https://github.com/cilium/cilium)
- [BPF ringbuf library](https://github.com/xdp-project/bpf-ringbuf)