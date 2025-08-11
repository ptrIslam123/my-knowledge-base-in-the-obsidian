**Transmit Queue = TX Queue** - это одно и то же.


## Описание реализации функции `ixgbe_device_add_output`

```c
struct ixgbe_device {
	volatile uint32_t* restrict addr;
	bool rx_enabled;
	bool tx_enabled;
};

struct ixgbe_descriptor {
	uint64_t addr; // Адрес буфера для пакета
	uint64_t metadata; // мета информация по дескриптору
};

struct ixgbe_transmit_head {
	uint32_t value;
	uint8_t _padding[60];
} __attribute__((packed));

static inline bool ixgbe_device_add_output(
	struct ixgbe_device* device, 
	volatile struct ixgbe_descriptor* ring,
	volatile struct ixgbe_transmit_head* transmit_head, 
	volatile uint32_t* restrict* out_tail_addr
);
```


Функция `ixgbe_device_add_output` настраивает **очередь передачи (TX queue)** сетевого контроллера Intel 82599ES. Эта очередь отвечает за отправку исходящих сетевых пакетов.

### **1. Поиск свободной transmit queue**
```c
uint32_t queue_index = 0;
for (; queue_index < TRANSMIT_QUEUES_COUNT; queue_index++) {
    if (reg_is_field_cleared(device->addr, REG_TXDCTL(queue_index), REG_TXDCTL_ENABLE)) {
        break;
    }
}
```
**Что делает**: Ищет первую отключенную очередь передачи (бит `TXDCTL.ENABLE` сброшен)
**Зачем**: Контроллер поддерживает несколько очередей передачи, нужно найти свободную

### **2. Настройка Descriptor Ring**
```c
uintptr_t ring_phys_addr = tn_mem_virt_to_phys((void*) ring);
reg_write(device->addr, REG_TDBAH(queue_index), (uint32_t)(ring_phys_addr >> 32));
reg_write(device->addr, REG_TDBAL(queue_index), (uint32_t) ring_phys_addr);
```
**Что делает**: Устанавливает физический адрес кольцевого буфера дескрипторов
- `TDBAL` - младшие 32 бита адреса
- `TDBAH` - старшие 32 бита адреса (для 64-битной адресации)
**Требование**: Адрес должен быть выровнен по 128 байт

### **3. Установка размера Descriptor Ring**
```c
reg_write(device->addr, REG_TDLEN(queue_index), IXGBE_RING_SIZE * sizeof(struct ixgbe_descriptor));
```
**Что делает**: Задает размер кольцевого буфера в байтах
- `IXGBE_RING_SIZE` - количество дескрипторов в кольце
- Умножается на размер одного дескриптора (16 байт)

### **4. Конфигурация параметров передачи (TXDCTL)**
```c
reg_write_field(device->addr, REG_TXDCTL(queue_index), REG_TXDCTL_PTHRESH, 60);
reg_write_field(device->addr, REG_TXDCTL(queue_index), REG_TXDCTL_HTHRESH, 4);
```
**Параметры**:
- `PTHRESH` (60) - Threshold for prefetching descriptors
- `HTHRESH` (4) - Threshold for host-based fetching
**Оптимизация**: Эти значения оптимизированы для производительности 10G трафика

### **5. Настройка Head Write-Back**
```c
uintptr_t head_phys_addr = tn_mem_virt_to_phys((void*) &(transmit_head->value));
if (head_phys_addr % 16u != 0) {
    TN_DEBUG("Transmit head's physical address is not aligned properly");
    return false;
}
reg_write(device->addr, REG_TDWBAH(queue_index), (uint32_t)(head_phys_addr >> 32));
reg_write(device->addr, REG_TDWBAL(queue_index), (uint32_t) head_phys_addr | 1u);
```
**Что делает**: Настраивает аппаратное обновление указателя головы
- **Head Write-Back** - аппаратное обновление позиции головы очереди
- **Требование выравнивания**: 16-байтное выравнивание (qword-aligned)
- **Бит 0**: `Head_WB_En` - включение write-back

### **6. Отключение Relaxed Ordering**
```c
reg_clear_field(device->addr, REG_DCATXCTRL(queue_index), REG_DCATXCTRL_TX_DESC_WB_RO_EN);
```
**Зачем**: Предотвращение обратного обновления указателя головы
**Безопасность**: Гарантирует корректную последовательность обновлений

### **7. Активация transmit path**
```c
if (!device->tx_enabled) {
    reg_set_field(device->addr, REG_DMATXCTL, REG_DMATXCTL_TE);
    device->tx_enabled = true;
}
```
**Что делает**: Включает весь transmit path контроллера
**Важно**: Выполняется только один раз для первой включаемой очереди

### **8. Активация очереди и ожидание**
```c
reg_set_field(device->addr, REG_TXDCTL(queue_index), REG_TXDCTL_ENABLE);
IF_AFTER_TIMEOUT(1000 * 1000, reg_is_field_cleared(device->addr, REG_TXDCTL(queue_index), REG_TXDCTL_ENABLE))
```
**Что делает**: Включает очередь и ожидает аппаратного подтверждения
**Таймаут**: 1 секунда на активацию

### **9. Возврат указателя на Tail Register**
```c
*out_tail_addr = device->addr + REG_TDT(queue_index);
```
**Важность**: Возвращает адрес регистра TDT для последующего управления очередью

## **Архитектура TX Queue**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Descriptor    │    │   Packet        │    │   Hardware      │
│   Ring          │    │   Buffers       │    │   Registers     │
│   [Software]    │    │   [Software]    │    │   [NIC]         │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│   Descriptor 0  │───▶│   Buffer 0      │    │   TDBAL/TDBAH   │
│   Descriptor 1  │───▶│   Buffer 1      │    │   TDLEN         │
│   ...           │    │   ...           │    │   TDT           │
│   Descriptor N  │───▶│   Buffer N      │    │   TXDCTL        │
└─────────────────┘    └─────────────────┘    └─────────────────┘
       ↑                      ↑                       ↑
       │                      │                       │
       └──────────────────────┼───────────────────────┘
                              │
                         ┌────┴────┐
                         │  DMA    │
                         │ Engine  │
                         └─────────┘
```

## **Процесс передачи пакета**

### **Со стороны Software**:
1. Заполняет дескрипторы данными пакетов
2. Обновляет Tail Pointer (TDT)
3. Ожидает завершения передачи

### **Со стороны Hardware**:
1. Видит обновленный TDT
2. Читает дескрипторы через DMA
3. Передает данные пакетов в сеть
4. Обновляет Head Pointer через write-back

## **Критически важные моменты**

### **Выравнивание памяти**:
- Descriptor ring должен быть выровнен по 128 байт
- Head write-back адрес должен быть выровнен по 16 байт

### **Порядок инициализации**:
1. Настройка дескрипторов и буферов
2. Установка регистров адреса/длины
3. Конфигурация параметров TXDCTL
4. Настройка head write-back
5. Активация transmit path
6. Активация очереди

### **Оптимизации производительности**:
- Prefetch threshold (PTHRESH) - предварительная выборка дескрипторов
- Host threshold (HTHRESH) - оптимизация доступа к хосту
- Head write-back - снижение нагрузки на CPU
