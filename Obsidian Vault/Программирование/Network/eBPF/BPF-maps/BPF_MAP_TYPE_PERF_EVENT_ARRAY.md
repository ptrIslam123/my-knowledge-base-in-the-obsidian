#### **1. `bpf_perf_event_output()` — отправка данных в perf_buffer**
**Назначение**:  
Основной способ отправки событий из eBPF в `perf_event_array`.

**Сигнатура**:
```c
long bpf_perf_event_output(void *ctx, struct bpf_map *map, u64 flags, void *data, u64 size);
```

**Параметры**:
- `ctx` — контекст программы (например, `struct xdp_md *` для XDP).
- `map` — указатель на `BPF_MAP_TYPE_PERF_EVENT_ARRAY`.
- `flags`:
  - `BPF_F_CURRENT_CPU` — отправлять на текущем CPU.
  - Или явный номер CPU (например, `2`).
- `data` — указатель на данные для отправки.
- `size` — размер данных.

**Пример** (отправка события при системном вызове):
```c
struct event {
    u32 pid;
    char comm[16];
};

struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} events SEC(".maps");

SEC("kprobe/sys_execve")
int handle_execve(void *ctx) {
    struct event e = {
        .pid = bpf_get_current_pid_tggid(),
    };
    bpf_get_current_comm(&e.comm, sizeof(e.comm));
    
    bpf_perf_event_output(ctx, &events, BPF_F_CURRENT_CPU, &e, sizeof(e));
    return 0;
}
```

**Как работает**:
1. Данные копируются в perf-буфер **ядра**.
2. Пользовательский процесс читает их через `perf_buffer__poll()`.

---

## **2. `bpf_perf_event_read()` — чтение счётчиков perf**
**Назначение**:  
Чтение значений hardware/software perf-счётчиков (например, CPU cycles, cache misses).

**Сигнатура**:
```c
long bpf_perf_event_read(struct bpf_map *map, u64 flags);
```

**Параметры**:
- `map` — `BPF_MAP_TYPE_PERF_EVENT_ARRAY` с открытыми perf-событиями.
- `flags` — номер CPU (или `BPF_F_CURRENT_CPU`).

**Пример** (чтение количества циклов CPU):
```c
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
    __uint(key_size, sizeof(u32));
    __uint(value_size, sizeof(u32));
} pmu_cycles SEC(".maps");

SEC("kprobe/sys_execve")
int handle_execve(void *ctx) {
    u64 cycles = bpf_perf_event_read(&pmu_cycles, BPF_F_CURRENT_CPU);
    bpf_printk("CPU cycles: %llu", cycles);
    return 0;
}
```

**Ограничения**:
- Требует предварительной настройки perf-событий из userspace.

---

## **3. `bpf_perf_event_read_value()` — чтение счётчиков с дополнительными метаданными**
**Назначение**:  
Чтение perf-счётчиков вместе с метаданными (например, временем включения/выключения).

**Сигнатура**:
```c
long bpf_perf_event_read_value(struct bpf_map *map, u64 flags, struct bpf_perf_event_value *buf, u32 buf_size);
```

**Параметры**:
- `buf` — структура для результата:
  ```c
  struct bpf_perf_event_value {
      u64 counter;
      u64 enabled;
      u64 running;
  };
  ```

**Пример**:
```c
struct bpf_perf_event_value val;
bpf_perf_event_read_value(&pmu_cycles, BPF_F_CURRENT_CPU, &val, sizeof(val));
bpf_printk("Counter: %llu, Enabled: %llu", val.counter, val.enabled);
```

---

## **4. `bpf_skb_output()` — отправка данных из сетевых программ (для sk_buff)**
**Назначение**:  
Аналог `bpf_perf_event_output()` для программ типа `sch_cls` (классификация трафика).

**Сигнатура**:
```c
long bpf_skb_output(void *ctx, struct bpf_map *map, u64 flags, void *data, u64 size);
```

**Отличие от `bpf_perf_event_output`**:
- Оптимизирован для работы с `struct __sk_buff` (например, в TC-программах).

---

## **5. `bpf_xdp_output()` — отправка данных из XDP-программ**
**Назначение**:  
Аналог `bpf_perf_event_output()` для XDP (eXpress Data Path).

**Сигнатура**:
```c
long bpf_xdp_output(void *ctx, struct bpf_map *map, u64 flags, void *data, u64 size);
```

**Пример** (отправка заголовков пакетов):
```c
struct {
    __uint(type, BPF_MAP_TYPE_PERF_EVENT_ARRAY);
} xdp_events SEC(".maps");

SEC("xdp")
int handle_xdp(struct xdp_md *ctx) {
    struct ethhdr eth;
    bpf_xdp_output(ctx, &xdp_events, BPF_F_CURRENT_CPU, &eth, sizeof(eth));
    return XDP_PASS;
}
```

---

## **Сравнение функций**
| Функция                      | Где используется          | Особенности                     |
|------------------------------|---------------------------|---------------------------------|
| `bpf_perf_event_output()`     | Все программы eBPF        | Основной метод для событий      |
| `bpf_perf_event_read()`       | Мониторинг perf-счётчиков | Только чтение, без данных       |
| `bpf_perf_event_read_value()` | Точные измерения perf     | + метаданные                   |
| `bpf_skb_output()`            | TC (traffic control)      | Оптимизирован для `__sk_buff`  |
| `bpf_xdp_output()`            | XDP                       | Оптимизирован для `xdp_md`     |

---

## **Когда использовать?**
1. **`bpf_perf_event_output`** — для передачи произвольных событий (трейсинг, логи).
2. **`bpf_perf_event_read*`** — для мониторинга производительности (CPU, cache).
3. **`bpf_{skb,xdp}_output`** — для сетевых программ (если нужна оптимизация).

---

### **Итог**
- **Для событий** → `bpf_perf_event_output`.
- **Для метрик** → `bpf_perf_event_read_value`.
- **Для сети** → `bpf_skb_output` / `bpf_xdp_output`. 

Все они работают с `BPF_MAP_TYPE_PERF_EVENT_ARRAY`, но решают разные задачи.