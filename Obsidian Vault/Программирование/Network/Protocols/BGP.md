**BGP — это протокол маршрутизации между автономными системами (AS).**
### **Ключевая аналогия:**
- **Внутри страны (AS):** Используются "внутренние" протоколы (OSPF, EIGRP, IS-IS) — как городской транспорт
- **Между странами (между AS):** Используется BGP — как международные авиарейсы

**Основная задача BGP:** Объявлять **достижимость сетей**, а не вычислять лучший путь (как  [[OSPF]]).

```
"Я, AS 64500, могу доставить трафик до сети 192.0.2.0/24"
А не:
"Иди через маршрутизатор A, потом B, потом C"
```

---

## **2. Ключевые концепции и термины**

### **A) Автономная система (AS)**
- Уникальный номер в интернете (1-65535 для 16-bit, сейчас 32-bit — до 4.3 млрд)
- Примеры:
  - **AS 15169** — Google
  - **AS 32934** — Facebook
  - **AS 56688** — Яндекс
- **Public ASN:** Глобально уникальные, для интернета
- **Private ASN (64512-65534):** Для внутреннего использования

### **B) Типы BGP сессий:**
1. **eBGP (external BGP):** Между разными AS
   - TTL = 1 (обычно, соседи должны быть напрямую соединены)
   - Административное расстояние: 20

2. **iBGP (internal BGP):** Внутри одной AS
   - TTL = 255
   - Административное расстояние: 200
   - **Правило синхронизации:** iBGP маршрут не используется, пока не узнан через IGP
   - **Full mesh requirement:** Все iBGP роутеры должны быть соединены (решается через Route Reflectors)

### **C) BGP соседи (Peers)**
- Устанавливают **TCP соединение на порт 179**
- Состояния (Finite State Machine):
  ```
  Idle → Connect → Active → OpenSent → OpenConfirm → Established
  ```
- **Только в состоянии Established** происходит обмен маршрутами

---

## **3. Формат BGP сообщений (4 типа)**

### **1. OPEN (Открытие сессии)**
```c
struct bgp_open {
    uint8_t version;      // Всегда 4
    uint16_t my_as;       // AS отправителя
    uint16_t hold_time;   // Время ожидания Keepalive (сек)
    uint32_t router_id;   // IPv4-адрес (обычно loopback)
    // Optional Parameters (Capabilities):
    // - Multiprotocol Extensions (IPv6, VPNv4)
    // - Route Refresh
    // - 4-byte AS numbers
    // - GR (Graceful Restart)
}
```

### **2. UPDATE (Передача маршрутов)**
**Самое важное сообщение!** Несет информацию о:
- **Withdrawn Routes:** Сети, которые стали недоступны
- **Path Attributes:** Атрибуты пути (см. ниже)
- **NLRI (Network Layer Reachability Information):** Сами префиксы

### **3. KEEPALIVE (Поддержание сессии)**
- Отправляются каждые `hold_time/3` секунд
- Пустое сообщение (19 байт заголовок)
- Если нет UPDATE/KEEPALIVE за `hold_time` → сессия рвется

### **4. NOTIFICATION (Ошибка)**
- При ошибке → отправляется NOTIFICATION → закрывается TCP сессия
- Коды ошибок:
  ```
  1 - Message Header Error
  2 - OPEN Message Error  
  3 - UPDATE Message Error
  4 - Hold Timer Expired
  5 - Finite State Machine Error
  6 - Cease (административное закрытие)
  ```

---

## **4. BGP Path Attributes (Атрибуты пути) — СЕРДЦЕ BGP**

Атрибуты определяют **какой путь выбрать** и **как передать маршрут дальше**.

### **Классификация атрибутов:**

#### **A) Well-known mandatory (Должны быть в каждом UPDATE)**
1. **ORIGIN (Код 1):** Откуда пришел маршрут
   - `IGP` (0) — из внутреннего протокола (сеть в этой AS)
   - `EGP` (1) — через старый протокол EGP (устарел)
   - `INCOMPLETE` (2) — перераспределен (redistributed)

2. **AS_PATH (Код 2):** Список пройденных AS
   ```python
   # Для eBGP: добавляется своя AS в начало
   AS_PATH = [AS64500, AS64499, AS64498]
   # Для iBGP: AS_PATH не меняется!
   ```

3. **NEXT_HOP (Код 3):** IP-адрес следующего прыжка
   - **eBGP:** Адрес соседнего роутера
   - **iBGP:** Обычно не меняется (сохраняется оригинальный NEXT_HOP)

#### **B) Well-known discretionary (Могут быть, но все понимают)**
4. **LOCAL_PREF (Код 5):** **Главный атрибут для выбора исходящего трафика**
   - Только для iBGP
   - Чем выше (0-4294967295), тем лучше
   - По умолчанию: 100
   ```python
   # Пример: предпочитать канал через Google
   if next_hop == google_peer:
       local_pref = 200
   elif next_hop == telecom_peer:
       local_pref = 150
   ```

5. **ATOMIC_AGGREGATE (Код 6):** "Я агрегировал маршруты, возможна потеря информации"

#### **C) Optional transitive (Могут игнорировать, но передают дальше)**
6. **AGGREGATOR (Код 7):** Кто сделал агрегацию (AS и IP)

7. **COMMUNITIES (Код 8):** **Метки для групповой политики**
   - Формат: `AS:NUMBER` (например, `64500:80`)
   - Значения:
     - `64500:666` — не анонсировать наружу
     - `64500:777` — не анонсировать никому
     - `0:80` — normal (стандартное коммьюнити)
     - Well-known communities:
       - `NO_EXPORT (0xFFFFFF01)` — не отправлять за пределы AS
       - `NO_ADVERTISE (0xFFFFFF02)` — не анонсировать никому
       - `NO_EXPORT_SUBCONFED (0xFFFFFF03)` — не отправлять другим sub-AS

#### **D) Optional non-transitive (Могут игнорировать и не передавать)**
8. **MED (Multi-Exit Discriminator, Код 4):** **"Подсказка" соседней AS о входном пути**
   - Чем ниже MED, тем лучше
   - Действует только между соседними AS
   - По умолчанию: 0
   ```python
   # AS 64500 говорит AS 64501:
   # "Заходи ко мне через канал A (MED=50), а не через B (MED=100)"
   ```

9. **ORIGINATOR_ID (Код 9):** Для Route Reflectors
10. **CLUSTER_LIST (Код 10):** Для Route Reflectors

---

## **5. Процесс принятия решений BGP (BGP Decision Process)**

**Когда приходит несколько путей к одной сети, BGP выбирает по порядку:**

### **12-шаговый алгоритм (Cisco):**
1. **Highest Weight** (Cisco-specific, локальный)
2. **Highest LOCAL_PREF**
3. **Local routes** (те, что сгенерированы на этом роутере)
4. **Shortest AS_PATH** (только длина, не содержание)
5. **Lowest ORIGIN** (IGP < EGP < INCOMPLETE)
6. **Lowest MED**
7. **eBGP over iBGP**
8. **Lowest IGP metric to NEXT_HOP**
9. **Oldest path** (для eBGP)
10. **Lowest Router ID**
11. **Minimum Cluster List length**
12. **Lowest Neighbor Address**

**В реальности:** Чаще всего решает **LOCAL_PREF** или **AS_PATH**.

---

## **6. BGP Finite State Machine (Для эмуляции в "Пересвет-СТ")**

```c
// Упрощенная реализация состояний
enum bgp_state {
    BGP_STATE_IDLE = 0,        // Начальное состояние
    BGP_STATE_CONNECT,         // Пытаемся установить TCP
    BGP_STATE_ACTIVE,          // Не удалось, ждем retry timer
    BGP_STATE_OPENSENT,        // Отправили OPEN, ждем ответ
    BGP_STATE_OPENCONFIRM,     // Получили OPEN, отправили KEEPALIVE
    BGP_STATE_ESTABLISHED      // Рабочее состояние
};

// События:
enum bgp_event {
    BGP_EVENT_START,           // Админ запустил BGP
    BGP_EVENT_STOP,            // Админ остановил BGP
    BGP_EVENT_TCP_CONNECTED,   // TCP соединение установлено
    BGP_EVENT_TCP_FAILED,      // TCP ошибка
    BGP_EVENT_OPEN_RECEIVED,   // Получили OPEN сообщение
    BGP_EVENT_KEEPALIVE_RECV,  // Получили KEEPALIVE
    BGP_EVENT_UPDATE_RECV,     // Получили UPDATE
    BGP_EVENT_NOTIFICATION,    // Получили NOTIFICATION
    BGP_EVENT_HOLD_TIMER_EXP,  // Таймер Hold Timer истек
    BGP_EVENT_KEEPALIVE_TIMER  // Пора отправить KEEPALIVE
};
```

---

## **7. Типы BGP сообщений в UPDATE**

### **A) Withdrawn Routes (Отзыв маршрутов)**
```
Length (1-2 byte) | Prefix (variable)
```
- Длина префикса в битах (от 0 до 32 для IPv4)
- Пример отзыва `10.0.0.0/8`:
  ```python
  withdrawn = struct.pack("!B", 8) + bytes([10, 0, 0, 0])[:1]
  # Только первый октет, т.к. 8 бит = 1 байт
  ```

### **B) Path Attributes**
```
Attribute Flags (1 byte) | Attribute Type Code (1 byte) | Length | Value
```
- **Флаги:**
  - Бит 0: Optional (0=well-known, 1=optional)
  - Бит 1: Transitive (0=non-transitive, 1=transitive)
  - Бит 2: Partial (0=complete, 1=partial)
  - Бит 3: Extended Length (0=1 байт длина, 1=2 байта)
  - Бит 4-7: Зарезервированы

### **C) NLRI (Анонсируемые префиксы)**
Аналогично Withdrawn Routes, но для анонсов.

---

## **8. BGP Extended Messages (RFC 8654)**

Для поддержки больших UPDATE (более 4096 префиксов):
- Новая максимальная длина: 65535 байт
- Флаг в OPEN: `Extended Message (Capability Code 6)`

---

## **9. Multiprotocol BGP (MP-BGP) Extensions**

BGP может переносить не только IPv4:
```c
// MP_REACH_NLRI (Код 14) и MP_UNREACH_NLRI (Код 15)
struct mp_nlri {
    uint16_t afi;        // Address Family Identifier
    uint8_t  safi;       // Subsequent Address Family Identifier
    uint8_t  next_hop_len;
    uint8_t  next_hop[/* переменная длина */];
    // ... NLRI
};

// Примеры AFI/SAFI:
// IPv4 Unicast:      AFI=1, SAFI=1
// IPv4 Multicast:    AFI=1, SAFI=2  
// IPv6 Unicast:      AFI=2, SAFI=1
// VPNv4:             AFI=1, SAFI=128
// EVPN:              AFI=25, SAFI=70
// SR Policy:         AFI=1, SAFI=73
```

---

## **10. BGP Security (Для тестирования устройств безопасности)**

### **Угрозы:**
1. **Prefix Hijacking:** Подделка анонса чужого префикса
   - 2008: Pakistan Telecom "украл" YouTube
   - 2014: Ростелеком перехватил трафик Visa/MasterCard

2. **Path Forgery:** Подделка AS_PATH

3. **Session Hijacking:** Перехват BGP сессии

### **Защита:**
1. **RPKI (Resource Public Key Infrastructure):**
   - ROA (Route Origin Authorization): Кто имеет право анонсировать префикс
   - ROV (Route Origin Validation): Проверка на маршрутизаторе

2. **BGPsec:** Подпись AS_PATH (очень сложно внедряется)

3. **Фильтрация:**
   - **prefix-lists:** Какие префиксы принимать
   - **as-path access-lists:** Регулярные выражения для AS_PATH
   - **route-maps:** Комплексная политика

---

## **11. Пример BGP диалога (для эмуляции в "Пересвет-СТ")**

```python
# 1. Установка TCP (роутер A: 10.0.0.1, роутер B: 10.0.0.2)
A → B: TCP SYN (порт 179)
B → A: TCP SYN-ACK
A → B: TCP ACK

# 2. BGP OPEN обмен
A → B: BGP OPEN (version=4, AS=64500, hold=180, router_id=1.1.1.1)
B → A: BGP OPEN (version=4, AS=64501, hold=180, router_id=2.2.2.2)

# 3. KEEPALIVE
A → B: BGP KEEPALIVE
B → A: BGP KEEPALIVE
# Теперь состояние ESTABLISHED

# 4. Анонс маршрута
A → B: BGP UPDATE(
    path_attributes=[
        ORIGIN=IGP,
        AS_PATH=[64500],
        NEXT_HOP=10.0.0.1,
        LOCAL_PREF=100
    ],
    nlri=[192.0.2.0/24]
)

# 5. Ежесекундные KEEPALIVE
# каждые 60 сек (180/3)
```

---

## **12. Зачем BGP в "Пересвет-СТ"?**

Для тестирования сетевых устройств нужно эмулировать:

### **Сценарии тестирования:**
1. **Устойчивость к flap (мерцанию маршрутов):**
   ```python
   # Быстрое объявление/отзыв маршрута
   for i in range(1000):
       send_update(prefix="203.0.113.0/24")
       sleep(0.1)
       send_withdraw(prefix="203.0.113.0/24")
       sleep(0.1)
   # Проверяем, не упадет ли устройство
   ```

2. **Stress test таблицы маршрутизации:**
   ```python
   # Анонсируем полную таблицу интернета
   # ~900,000 маршрутов IPv4, ~100,000 IPv6
   for prefix in full_bgp_table:
       send_update(prefix=prefix, as_path=random_as_path())
   ```

3. **Атаки и защита:**
   - **Prefix hijacking:** Анонс более специфичного префикса
   - **AS_PATH атака:** Очень длинный AS_PATH (максимум 255 AS)
   - **COMMUNITIES overflow:** Очень много коммьюнити в одном UPDATE

4. **Тестирование политик:**
   ```python
   # Проверяем фильтрацию
   send_update(prefix="192.0.2.0/24", as_path=[64500])
   # Должен принять
   
   send_update(prefix="192.0.2.0/24", as_path=[64513])  # Private AS
   # Должен отфильтровать
   ```

---

## **13. Реализация BGP в DPDK-приложении**

### **Архитектура для "Пересвет-СТ":**
```c
struct bgp_session {
    uint32_t router_id;
    uint16_t local_as;
    uint16_t remote_as;
    uint32_t remote_ip;
    
    enum bgp_state state;
    uint32_t hold_time;
    uint32_t keepalive_timer;
    
    // TCP сокет (или raw через DPDK)
    int tcp_fd;
    
    // Очереди сообщений
    struct rte_ring *tx_queue;
    struct rte_ring *rx_queue;
    
    // Таблица маршрутов
    struct bgp_route_table *rib;
    struct bgp_route_table *loc_rib;
};

// Обработчик BGP в DPDK
void bgp_process(struct bgp_session *session) {
    switch (session->state) {
    case BGP_STATE_IDLE:
        // Инициируем TCP соединение через DPDK
        start_tcp_connection(session->remote_ip, 179);
        session->state = BGP_STATE_CONNECT;
        break;
        
    case BGP_STATE_ESTABLISHED:
        // Проверяем таймеры
        if (keepalive_timer_expired()) {
            send_bgp_keepalive(session);
            reset_keepalive_timer();
        }
        
        // Обрабатываем входящие сообщения
        struct bgp_message *msg;
        if (rte_ring_dequeue(session->rx_queue, (void**)&msg) == 0) {
            process_bgp_message(session, msg);
            rte_mempool_put(bgp_msg_pool, msg);
        }
        break;
    }
}
```

### **Оптимизации для high-performance:**
1. **Batching сообщений:** Обрабатывать несколько UPDATE за один вызов
2. **Lockless структуры данных:** Для RIB (Routing Information Base)
3. **Преаллокация:** BGP сообщений в mempool
4. **Zero-copy:** Прямая передача BGP сообщений между потоками через rte_ring

---

## **14. Вопросы на собеседовании**

### **Вопрос 1: "Чем eBGP отличается от iBGP?"**
**Ответ:** "eBGP — между разными AS, TTL=1, ADMIN distance=20, добавляет свою AS в AS_PATH. iBGP — внутри одной AS, TTL=255, ADMIN distance=200, не меняет AS_PATH, требует full mesh или Route Reflectors."

### **Вопрос 2: "Как BGP выбирает лучший путь?"**
**Ответ:** "По 12-шаговому алгоритму. Ключевые этапы: LOCAL_PREF (чем выше, тем лучше), затем AS_PATH (чем короче, тем лучше), затем ORIGIN (IGP лучше EGP), затем MED (чем ниже, тем лучше). В 90% случаев решает LOCAL_PREF."

### **Вопрос 3: "Что такое BGP Communities и зачем они?"**
**Ответ:** "Это метки формата AS:VALUE для группового применения политик. Например, 64500:666 — не анонсировать наружу. Позволяют управлять маршрутами группами без конфигурации каждого префикса отдельно."

### **Вопрос 4: "Как бы вы эмулировали BGP для тестирования маршрутизатора?"**
**Ответ:** "Реализовал бы state machine BGP, отправлял бы полную таблицу интернета (900k+ маршрутов), тестировал бы устойчивость к route flap, проверял бы фильтрацию по prefix-list/AS-path/community, и измерял бы время сходимости при изменениях."

### **Вопрос 5: "Что такое BGP hijacking и как защититься?"**
**Ответ:** "Это анонс чужих префиксов. Защита: RPKI/ROV для проверки легитимности, фильтрация по max-prefix limit, фильтрация private AS, использование AS-path filters."

---

## **15. Ресурсы для углубления**

1. **RFC 4271** — BGP-4 (основной)
2. **RFC 4456** — BGP Route Reflection
3. **RFC 4760** — Multiprotocol Extensions
4. **RFC 5492** — Capabilities Advertisement
5. **RFC 6811** — BGP Prefix Origin Validation (RPKI)
6. **BGP Looking Glass** — реальные таблицы маршрутизации
7. **RIPE RIS** — Raw BGP данные

**Для "Пересвет-СТ":** Вам нужно будет эмулировать не только vanilla BGP, но и расширения (VPNv4, EVPN, SR Policy), а также генерировать реалистичный "плохой" трафик для тестирования устройств безопасности.

BGP — это сложно, но именно его сложность делает интернет гибким и масштабируемым! Удачи в подготовке!