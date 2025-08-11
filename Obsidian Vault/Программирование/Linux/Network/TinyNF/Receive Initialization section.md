**Receive Queue = RX Queue** - это одно и то же.
## Описание реализации функции `ixgbe_device_set_input`

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

static inline bool ixgbe_device_set_input(
	struct ixgbe_device* device, 
	volatile struct ixgbe_descriptor* ring, 
	volatile uint32_t* restrict* out_tail_addr
);
```


Этот код настраивает **Receive Queue (очередь приема)** сетевого контроллера Intel 82599ES. 
Тобишь эта функция превращает "сырую" сетевую карту в готовую к приему трафика систему


## **Архитектура Receive Queue**

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Descriptor    │    │   Packet        │    │   Hardware      │
│   Ring          │    │   Buffers       │    │   Registers     │
│   [Software]    │    │   [Software]    │    │   [NIC]         │
├─────────────────┤    ├─────────────────┤    ├─────────────────┤
│   Descriptor 0  │───▶│   Buffer 0      │    │   RDBAL/RDBAH   │
│   Descriptor 1  │───▶│   Buffer 1      │    │   RDLEN         │
│   ...           │    │   ...           │    │   RDT           │
│   Descriptor N  │───▶│   Buffer N      │    │   RXDCTL        │
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

## **Что происходит при приеме пакета**:

##### **1. Прием пакета из сети**
- **PHY** принимает биты из кабеля/SFP+
- **MAC** обрабатывает кадр Ethernet
- **Фильтры** проверяют: наш ли это пакет?
- Если пакет подходит → передается в **DMA Engine**
##### **2. Поиск свободного дескриптора**
##### **3. DMA запись пакета в память**
##### **4. Обновление дескриптора**
##### **5. Прерывание или опрос(polling)**

# Примерная реализация
### **1. Проверка активности очереди**
```c
if (!reg_is_field_cleared(device->addr, REG_RXDCTL(queue_index), REG_RXDCTL_ENABLE)) {
    return false;
}
```
**Проверяет**: Что очередь уже включена (бит `RXDCTL.ENABLE` сброшен)
**Зачем**: Предотвращение повторной инициализации активной очереди

### **2. Настройка Descriptor Ring**
```c
uintptr_t ring_phys_addr = tn_mem_virt_to_phys((void*) ring);
reg_write(device->addr, REG_RDBAH(queue_index), (uint32_t)(ring_phys_addr >> 32));
reg_write(device->addr, REG_RDBAL(queue_index), (uint32_t) ring_phys_addr);
```
**Что делает**: Устанавливает физический адрес кольцевого буфера дескрипторов
- `RDBAL` - младшие 32 бита адреса
- `RDBAH` - старшие 32 бита адреса (для 64-битной адресации)

### **3. Установка размера Descriptor Ring**
```c
reg_write(device->addr, REG_RDLEN(queue_index), IXGBE_RING_SIZE * sizeof(struct ixgbe_descriptor));
```
**Что делает**: Задает размер кольцевого буфера в байтах
- `IXGBE_RING_SIZE` - количество дескрипторов в кольце
- Умножается на размер одного дескриптора

### **4. Конфигурация параметров приема (SRRCTL)**
```c
reg_write_field(device->addr, REG_SRRCTL(queue_index), REG_SRRCTL_BSIZEPACKET, 
               sizeof(struct ixgbe_packet_data) / 1024u);
reg_set_field(device->addr, REG_SRRCTL(queue_index), REG_SRRCTL_DROP_EN);
```
**Параметры**:
- `BSIZEPACKET` - размер буфера для пакетов (в KB)
- `DROP_EN` - включение отбрасывания пакетов при заполнении очереди

### **5. Активация очереди**
```c
reg_set_field(device->addr, REG_RXDCTL(queue_index), REG_RXDCTL_ENABLE);
IF_AFTER_TIMEOUT(1000 * 1000, reg_is_field_cleared(device->addr, REG_RXDCTL(queue_index), REG_RXDCTL_ENABLE))
```
**Что делает**: Включает очередь и ожидает аппаратного подтверждения
**Зачем**: Гарантия что очередь готова к работе перед использованием

### **6. Инициализация Tail Pointer**
```c
reg_write(device->addr, REG_RDT(queue_index), IXGBE_RING_SIZE - 1u);
```
**Что делает**: Устанавливает указатель хвоста на последний валидный дескриптор
**Важность**: Указывает hardware с какого дескриптора начинать обработку

### **7. Активация всего receive path**
```c
if (!device->rx_enabled) {
    // Комплексная активация приемного тракта
    reg_set_field(device->addr, REG_SECRXCTRL, REG_SECRXCTRL_RX_DIS);
    // ... ожидание готовности ...
    reg_set_field(device->addr, REG_RXCTRL, REG_RXCTRL_RXEN);
    reg_clear_field(device->addr, REG_SECRXCTRL, REG_SECRXCTRL_RX_DIS);
    reg_set_field(device->addr, REG_CTRLEXT, REG_CTRLEXT_NSDIS);
    device->rx_enabled = true;
}
```
**Многоэтапный процесс**:
1. Остановка receive path
2. Ожидание опустошения буферов
3. Включение receive engine
4. Настройка No Snoop для производительности

### **8. DCA (Direct Cache Access) настройка**
```c
reg_clear_field(device->addr, REG_DCARXCTRL(queue_index), REG_DCARXCTRL_UNKNOWN);
```
**Что делает**: Настраивает оптимизацию доступа к кэшу для receive очереди

### **9. Возврат указателя на Tail Register**
```c
*out_tail_addr = device->addr + REG_RDT(queue_index);
```
**Важность**: Возвращает адрес регистра RDT для последующего управления очередью
