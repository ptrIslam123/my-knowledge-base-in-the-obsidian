## Основные драйверы сетевых устройств (find dpdk | grep ethdev.c):

### Драйверы Intel:
- `./drivers/net/intel/e1000/em_ethdev.c` - Intel e1000 (82540, 82545, 82546)
- `./drivers/net/intel/e1000/igb_ethdev.c` - Intel e1000 (82575, 82576, 82580)
- `./drivers/net/intel/e1000/igc_ethdev.c` - Intel I225/L225 (2.5GbE)
- `./drivers/net/intel/ixgbe/ixgbe_ethdev.c` - Intel 10GbE (82599, X520, X540)
- `./drivers/net/intel/i40e/i40e_ethdev.c` - Intel 40GbE (XL710, X710)
- `./drivers/net/intel/ice/ice_ethdev.c` - Intel E800 series (100GbE)
- `./drivers/net/intel/iavf/iavf_ethdev.c` - Intel Virtual Function

### Драйверы других производителей:
- `./drivers/net/virtio/virtio_ethdev.c` - Virtio (виртуальные устройства)
- `./drivers/net/mlx5/mlx5_ethdev.c` - Mellanox ConnectX-4/5/6
- `./drivers/net/ena/ena_ethdev.c` - Amazon ENA
- `./drivers/net/bnxt/bnxt_ethdev.c` - Broadcom NetXtreme
- `./drivers/net/cxgbe/cxgbe_ethdev.c` - Chelsio T5/T6

### Драйверы для виртуализации:
- `./drivers/net/virtio/virtio_user_ethdev.c` - Virtio пользовательский режим
- `./drivers/net/virtio/virtio_pci_ethdev.c` - Virtio PCI

## Что НЕ является драйверами устройств:

- `./lib/ethdev/rte_ethdev.c` - это **общий API** ethdev, а не драйвер
- `./lib/ethdev/rte_ethdev_cman.c` - управление congestion management

## Рассмотрим следуйщий код:

Начните с изучения одного из популярных драйверов:

1. **virtio** - хорош для понимания виртуализации:
   ```bash
   ls ./drivers/net/virtio/
   # Основные файлы: virtio_ethdev.c, virtio_rxtx.c
   ```

2. **ixgbe** - классический 10GbE драйвер:
   ```bash
   ls ./drivers/net/intel/ixgbe/
   # Основные файлы: ixgbe_ethdev.c, ixgbe_rxtx.c, ixgbe_ethdev.h
   ```


# Разбор virtio_ethdev.h
### **Константы ограничений**
```c
#define VIRTIO_MAX_RX_QUEUES 128U
#define VIRTIO_MAX_TX_QUEUES 128U
#define VIRTIO_MAX_MAC_ADDRS 64
#define VIRTIO_MIN_RX_BUFSIZE 64
#define VIRTIO_MAX_RX_PKTLEN  9728U
```
- Максимальное количество очередей RX/TX
- Лимиты на размеры буферов и пакетов

### **Флаги возможностей (Features) - ВАЖНО!**
```c
#define VIRTIO_PMD_DEFAULT_GUEST_FEATURES	
	(1u << VIRTIO_NET_F_MAC		  |	
	 1u << VIRTIO_NET_F_STATUS	  |	
	 1u << VIRTIO_NET_F_MQ		  |	
	 // ... и многие другие
```
Это **ключевая часть**! Здесь определяются возможности, которые поддерживает драйвер:

- **VIRTIO_NET_F_MAC** - устройство предоставляет MAC адрес
- **VIRTIO_NET_F_STATUS** - информация о статусе ссылки
- **VIRTIO_NET_F_MQ** - multi-queue (многоочередность)
- **VIRTIO_NET_F_MRG_RXBUF** - объединение RX буферов
- **VIRTIO_F_VERSION_1** - поддержка Virtio 1.0+ спецификации
- **VIRTIO_F_RING_PACKED** - packed rings (новый формат дескрипторов)

### **Декларация eth_dev_ops**

**`struct eth_dev_ops`** - это одна из самых важных структур в DPDK, определяющая **интерфейс между фреймворком и драйвером**.

Это структура, содержащая **указатели на функции** (function pointers), которые драйвер должен реализовать. DPDK framework вызывает эти функции для управления сетевым устройством.

## Основные группы операций в `eth_dev_ops`:

### 1. **Управление устройством**
- `dev_configure` - конфигурация устройства
- `dev_start` - запуск устройства
- `dev_stop` - остановка устройства  
- `dev_close` - закрытие устройства
- `dev_reset` - сброс устройства

### 2. **Управление очередями**
- `rx_queue_setup` - настройка RX очереди
- `tx_queue_setup` - настройка TX очереди
- `rx_queue_release` - освобождение RX очереди
- `tx_queue_release` - освобождение TX очереди

### 3. **Статистика и информация**
- `stats_get` - получение статистики
- `stats_reset` - сброс статистики
- `link_update` - обновление статуса ссылки
- `info_get` - получение информации об устройстве

### 4. **Управление фильтрами**
- `mac_addr_add/remove` - управление MAC адресами
- `vlan_filter_set` - настройка VLAN фильтров
- `flow_ctrl_set` - управление flow control

## Пример реализации в Virtio:

```c
static const struct eth_dev_ops virtio_eth_dev_ops = {
    // Управление устройством
    .dev_configure = virtio_dev_configure,
    .dev_start = virtio_dev_start,
    .dev_stop = virtio_dev_stop,
    .dev_close = virtio_dev_close,
    
    // Управление очередями
    .rx_queue_setup = virtio_dev_rx_queue_setup,
    .tx_queue_setup = virtio_dev_tx_queue_setup,
    .rx_queue_release = virtio_dev_rx_queue_release,
    .tx_queue_release = virtio_dev_tx_queue_release,
    
    // Статистика
    .stats_get = virtio_dev_stats_get,
    .stats_reset = virtio_dev_stats_reset,
    .link_update = virtio_dev_link_update,
    
    // Фильтры
    .mac_addr_add = virtio_dev_mac_addr_add,
    .mac_addr_remove = virtio_dev_mac_addr_remove,
    
    // Прием/передача
    .rx_queue_intr_enable = virtio_dev_rx_queue_intr_enable,
    .rx_queue_intr_disable = virtio_dev_rx_queue_intr_disable,
};
```


## Как это используется в DPDK:

1. Драйвер регистрирует свою структуру `eth_dev_ops`
2. При создании устройства DPDK сохраняет указатель на эту структуру
3. Все вызовы API DPDK перенаправляются в соответствующие функции драйвера

```c
// Где-то в коде DPDK:
eth_dev->dev_ops = &virtio_eth_dev_ops;

// При вызове rte_eth_dev_start():
eth_dev->dev_ops->dev_start(eth_dev);
```

Это brilliant архитектурное решение, которое делает DPDK таким гибким и расширяемым!

## **Декларация публичных функции драйвера**

## 1. **Control Queue (CQ) функция**

```c
void virtio_dev_cq_start(struct rte_eth_dev *dev);
```
- **Назначение**: Запуск Control Queue для управления устройством
- **Параметры**:
  - `dev` - указатель на Ethernet устройство DPDK
- **Использование**: Для обработки контрольных сообщений (MAC changes, статус ссылки)

## 2. **RX Queue Setup функции**

```c
int virtio_dev_rx_queue_setup(struct rte_eth_dev *dev, 
                             uint16_t rx_queue_id,
                             uint16_t nb_rx_desc, 
                             unsigned int socket_id,
                             const struct rte_eth_rxconf *rx_conf,
                             struct rte_mempool *mb_pool);
```
- **Назначение**: Настройка RX очереди для приема пакетов
- **Параметры**:
  - `dev` - Ethernet устройство
  - `rx_queue_id` - ID очереди (0..VIRTIO_MAX_RX_QUEUES-1)
  - `nb_rx_desc` - количество дескрипторов в очереди
  - `socket_id` - NUMA сокет для выделения памяти
  - `rx_conf` - конфигурация RX (offloads, threshold)
  - `mb_pool` - пул памяти для буферов пакетов

```c
int virtio_dev_rx_queue_setup_finish(struct rte_eth_dev *dev,
                                   uint16_t rx_queue_id);
```
- **Назначение**: Завершение настройки RX очереди после базовой инициализации
- **Использование**: Настройка виртуального устройства Virtio

## 3. **TX Queue Setup функции**

```c
int virtio_dev_tx_queue_setup(struct rte_eth_dev *dev,
                             uint16_t tx_queue_id,
                             uint16_t nb_tx_desc,
                             unsigned int socket_id,
                             const struct rte_eth_txconf *tx_conf);
```
- **Назначение**: Настройка TX очереди для передачи пакетов
- **Параметры**
  -  `dev` - Ethernet устройство
  - `tx_queue_id` - ID очереди (0..VIRTIO_MAX_TX_QUEUES-1)
  - `nb_tx_desc` - количество дескрипторов в очереди
  - `socket_id` - NUMA сокет для выделения памяти
  - `rx_conf` - конфигурация RX (offloads, threshold)

```c
int virtio_dev_tx_queue_setup_finish(struct rte_eth_dev *dev,
                                   uint16_t tx_queue_id);
```
- **Назначение**: Завершение настройки TX очереди

## 4. **RX Functions (Прием пакетов)**

### Базовые функции:
```c
uint16_t virtio_recv_pkts(void *rx_queue, 
                         struct rte_mbuf **rx_pkts,
                         uint16_t nb_pkts);
```
- **Назначение**: Основная функция приема пакетов (split rings)
- **Параметры**:
  - `rx_queue` - указатель на RX очередь
  - `rx_pkts` - массив для принятых пакетов
  - `nb_pkts` - максимальное количество пакетов для приема
- **Возвращает**: количество фактически принятых пакетов

### Packed rings версии:
```c
uint16_t virtio_recv_pkts_packed(void *rx_queue, struct rte_mbuf **rx_pkts, uint16_t nb_pkts);
```
- **Назначение**: Прием для packed rings (более эффективный формат)

### Mergeable buffers версии:
```c
uint16_t virtio_recv_mergeable_pkts(void *rx_queue, ...);
uint16_t virtio_recv_mergeable_pkts_packed(void *rx_queue, ...);
```
- **Назначение**: Прием с объединением буферов (для больших пакетов)
- **Использование**: Когда пакет не помещается в один буфер

### In-order обработка:
```c
uint16_t virtio_recv_pkts_inorder(void *rx_queue, ...);
```
- **Назначение**: Прием с сохранением порядка пакетов
- **Использование**: Для определенных сценариев производительности

## 5. **TX Functions (Передача пакетов)**

```c
uint16_t virtio_xmit_pkts_prepare(void *tx_queue,
                                struct rte_mbuf **tx_pkts,
                                uint16_t nb_pkts);
```
- **Назначение**: Подготовка пакетов к передаче (предварительная обработка)

```c
uint16_t virtio_xmit_pkts(void *tx_queue,
                         struct rte_mbuf **tx_pkts,
                         uint16_t nb_pkts);
```
- **Назначение**: Основная функция передачи пакетов

```c
uint16_t virtio_xmit_pkts_packed(void *tx_queue, ...);
uint16_t virtio_xmit_pkts_inorder(void *tx_queue, ...);
```
- **Назначение**: Специализированные версии передачи


## 7. **Device Management**

```c
int eth_virtio_dev_init(struct rte_eth_dev *eth_dev);
```
- **Назначение**: Инициализация Virtio устройства
- **Использование**: Вызывается при создании устройства

```c
void virtio_interrupt_handler(void *param);
```
- **Назначение**: Обработчик прерываний от устройства
- **Параметры**: `param` - параметр (обычно очередь или устройство)

```c
int virtio_dev_stop(struct rte_eth_dev *dev);
int virtio_dev_close(struct rte_eth_dev *dev);
```
- **Назначение**: Остановка и закрытие устройства

## 8. **Utility Functions**

```c
bool virtio_rx_check_scatter(uint16_t max_rx_pkt_len,
                           uint16_t rx_buf_size,
                           bool rx_scatter_enabled,
                           const char **error);
```
- **Назначение**: Проверка возможности scatter RX (разброс приема)
- **Параметры**:
  - `max_rx_pkt_len` - максимальная длина пакета
  - `rx_buf_size` - размер RX буфера
  - `rx_scatter_enabled` - включен ли scatter mode
  - `error` - возвращает сообщение об ошибке

```c
uint16_t virtio_rx_mem_pool_buf_size(struct rte_mempool *mp);
```
- **Назначение**: Расчет эффективного размера буфера из mempool
- **Использование**: Для проверки достаточности размера буферов

## **Ключевые концепты:**

1. **Multiple Implementations**: Разные версии для разных сценариев
2. **Performance Optimization**: Векторизованные и специализированные версии  
3. **Virtio Specific**: Поддержка различных режимов Virtio (packed, mergeable)
4. **Memory Management**: Интеграция с DPDK mempool для эффективного управления памятью

Этот интерфейс показывает всю мощь DPDK - единый API с оптимизированными реализациями для разных hardware и software сценариев!

### 7. **Ключевые концепты, которые здесь видны:**

1. **Многоочередность** - поддержка до 128 RX/TX очередей
2. **Разные реализации**:
   - Обычные функции (`virtio_recv_pkts`)
   - Векторные оптимизированные (`virtio_recv_pkts_vec`) 
   - Для packed rings (`*_packed`)
   - Для mergeable буферов (`*_mergeable`)
   - In-order обработка (`*_inorder`)

3. **Полный lifecycle устройства**:
   - Инициализация (`eth_virtio_dev_init`)
   - Настройка очередей (`*_queue_setup`)
   - Работа (`recv/xmit_pkts`)
   - Остановка (`dev_stop`)
   - Закрытие (`dev_close`)

## Что дальше?
Этот файл дает нам карту всего драйвера. Теперь мы знаем:
- Какие возможности поддерживаются
- Какие основные функции реализованы
- Структуру драйвера

**Готов продолжить?** Можете скинуть начало `virtio_ethdev.c` чтобы посмотреть реализацию этих функций!