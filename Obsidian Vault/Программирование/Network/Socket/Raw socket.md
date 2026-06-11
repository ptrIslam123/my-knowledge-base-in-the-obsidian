**RAW-сокеты (сырые сокеты)** - это низкоуровневый интерфейс, позволяющий напрямую работать с сетевыми пакетами **без участия стека протоколов TCP/IP ядра**.  Пакетные сокеты используются для отправления или приема пакетов на уровне драйвера устройства (Уровень 2 OSI). Это позволяет пользователю реализовывать в пространстве пользователя протокольные модули над физическим уровнем. Вы сами формируете и парсите заголовки протоколов.

### **Аналогия:**
- **Обычные сокеты (TCP/UDP)** - как отправка письма через почту (указываете адрес, почта сама формирует конверт, марки и т.д.)
- **RAW-сокеты** - как изготовление конверта, марки, печати и почтовых отметок **своими руками**, полный контроль над каждым битом

## **Создание RAW-сокета**

```c
#include <sys/socket.h>
#include <netinet/in.h>
#include <linux/if_packet.h>
#include <net/if.h>
#include <arpa/inet.h>

// Разные уровни RAW-сокетов:

// Все Ethernet-кадры
int sock1 = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));

// Только IP пакеты
int sock2 = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_IP));

// Только ARP пакеты
int sock3 = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ARP));

// RAW IP (требует прав root)
int sock4 = socket(AF_INET, SOCK_RAW, IPPROTO_RAW);
```

## **Уровни сетевого стека и RAW-сокеты:**

```
┌─────────────────────────────────────────┐
│    Ваше приложение                      │
├─────────────────────────────────────────┤
│    RAW-сокет (AF_PACKET, SOCK_RAW)      │ ← Вы здесь!
├─────────────────────────────────────────┤
│    Ethernet кадр (MAC-адреса)           │
├─────────────────────────────────────────┤
│    Физический интерфейс (eth0, wlan0)   │
└─────────────────────────────────────────┘
```

---

## **Зачем использовать RAW-сокеты?**

### **1. Сетевые утилиты и диагностика:**
```c
// Примеры инструментов, использующих RAW-сокеты:
// - ping (ICMP)
// - traceroute
// - tcpdump/wireshark
// - nmap
// - arping
```

### **2. Кастомные протоколы:**
```c
// Можно реализовать свой протокол поверх Ethernet
struct my_protocol_header {
    uint16_t magic;     // 0xDEAD
    uint16_t version;   // 1
    uint32_t sequence;  // порядковый номер
    // ... свои поля
};
```

### **3. Сетевая безопасность:**
- IDS/IPS системы (обнаружение вторжений)
- Firewall (пакетная фильтрация)
- VPN реализации
- Сканеры уязвимостей

### **4. Высокопроизводительные сетевые приложения:**
- Пакетные брокеры
- Load balancers
- Кастомные роутеры

## ⚠️ **Ограничения и требования:**

### **Требуются права root:**
```bash
# RAW-сокеты требуют привилегий
sudo ./my_raw_socket_program
```

### **Ограничения в ядре Linux:**
```bash
# Проверка ограничений
cat /proc/sys/net/ipv4/ip_default_ttl
cat /proc/sys/net/ipv4/ip_forward

# Отключение защит (не рекомендуется в production!)
echo 0 > /proc/sys/net/ipv4/icmp_echo_ignore_all
```

## 🔄 **Сравнение с обычными сокетами:**

| Аспект | Обычные сокеты | RAW-сокеты |
|--------|----------------|------------|
| **Уровень** | Транспортный (TCP/UDP) | Канальный/Сетевой |
| **Контроль** | Ограниченный | Полный |
| **Производительность** | Высокая (ядро помогает) | Ниже (все в userspace) |
| **Сложность** | Низкая | Высокая |
| **Права** | Обычный пользователь | Требуется root |
| **Портируемость** | Высокая (POSIX) | Низкая (зависит от ОС) |

## 📝 **Практические примеры использования:**

### **Пример 1: Свой ping**
```c
// Создание ICMP Echo Request (ping)
int sock = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
// Формируем ICMP пакет вручную
// Отправляем через sendto()
// Ждем ответ через recvfrom()
```

### **Пример 2: ARP сканер**
```c
// Отправка ARP запросов для обнаружения хостов
int sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ARP));
// Формируем ARP "Who has IP X.X.X.X?"
// Анализируем ARP ответы
```

### **Пример 3: Пакетный генератор**
```c
// Генерация произвольного трафика для тестирования
for (int i = 0; i < 1000; i++) {
    create_random_packet(packet, &len);
    send_raw_packet_via_interface("eth0", packet, len);
}
```

---
## **Полный пример работы с RAW-сокетом**

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/ip.h>
#include <netinet/udp.h>
#include <linux/if_packet.h>
#include <net/if.h>
#include <arpa/inet.h>

#define BUFFER_SIZE 65536

// Структура Ethernet кадра
struct ethhdr {
    unsigned char h_dest[6];   // MAC назначения
    unsigned char h_source[6]; // MAC источника
    unsigned short h_proto;    // Протокол (ETH_P_IP, ETH_P_ARP и т.д.)
};

// Функция для отправки произвольного пакета через конкретный интерфейс
int send_raw_packet_via_interface(const char *iface_name, 
                                   const unsigned char *packet, 
                                   size_t packet_len) {
    // 1. Создаем RAW-сокет
    int sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (sock < 0) {
        perror("socket");
        return -1;
    }
    
    // 2. Получаем индекс интерфейса
    int ifindex = if_nametoindex(iface_name);
    if (ifindex == 0) {
        perror("if_nametoindex");
        close(sock);
        return -1;
    }
    
    // 3. Получаем MAC-адрес интерфейса
    struct ifreq ifr;
    memset(&ifr, 0, sizeof(ifr));
    strncpy(ifr.ifr_name, iface_name, IFNAMSIZ);
    
    if (ioctl(sock, SIOCGIFHWADDR, &ifr) < 0) {
        perror("ioctl SIOCGIFHWADDR");
        close(sock);
        return -1;
    }
    unsigned char *iface_mac = (unsigned char *)ifr.ifr_hwaddr.sa_data;
    
    // 4. Подготавливаем адрес для отправки
    struct sockaddr_ll sll = {
        .sll_family = AF_PACKET,
        .sll_ifindex = ifindex,          // Конкретный интерфейс!
        .sll_halen = ETH_ALEN,           // Длина MAC-адреса (6)
        .sll_protocol = htons(ETH_P_IP), // Протокол верхнего уровня
    };
    
    // Копируем MAC-адрес назначения (первые 6 байт пакета)
    memcpy(sll.sll_addr, packet, ETH_ALEN);
    
    // 5. Отправляем пакет
    ssize_t sent = sendto(sock, packet, packet_len, 0,
                         (struct sockaddr*)&sll, sizeof(sll));
    
    if (sent < 0) {
        perror("sendto");
        close(sock);
        return -1;
    }
    
    printf("Отправлено %zd байт через интерфейс %s\n", sent, iface_name);
    close(sock);
    return 0;
}

// Пример: создание кастомного UDP пакета
void create_custom_udp_packet(unsigned char *buffer, size_t *len) {
    struct ethhdr *eth = (struct ethhdr *)buffer;
    struct iphdr *ip = (struct iphdr *)(buffer + sizeof(struct ethhdr));
    struct udphdr *udp = (struct udphdr *)(buffer + sizeof(struct ethhdr) + sizeof(struct iphdr));
    char *payload = (char *)(buffer + sizeof(struct ethhdr) + sizeof(struct iphdr) + sizeof(struct udphdr));
    
    // 1. Ethernet заголовок
    memset(eth->h_dest, 0xFF, 6);          // Broadcast MAC
    memset(eth->h_source, 0x00, 6);        // Наш MAC (заполнится автоматически)
    eth->h_proto = htons(ETH_P_IP);        // Протокол IP
    
    // 2. IP заголовок
    ip->version = 4;                       // IPv4
    ip->ihl = 5;                           // Длина заголовка в 32-битных словах
    ip->tos = 0;                           // Type of Service
    ip->tot_len = htons(sizeof(struct iphdr) + sizeof(struct udphdr) + 10);
    ip->id = htons(12345);                 // Идентификатор пакета
    ip->frag_off = 0;                      // Фрагментация
    ip->ttl = 64;                          // Time To Live
    ip->protocol = IPPROTO_UDP;            // Протокол UDP
    ip->saddr = inet_addr("192.168.1.100"); // Source IP
    ip->daddr = inet_addr("192.168.1.1");  // Destination IP
    ip->check = 0;                         // Контрольная сумма (можно рассчитать)
    
    // 3. UDP заголовок
    udp->source = htons(12345);            // Source port
    udp->dest = htons(80);                 // Destination port
    udp->len = htons(sizeof(struct udphdr) + 10); // Длина UDP+данные
    udp->check = 0;                        // Контрольная сумма
    
    // 4. Полезная нагрузка
    strcpy(payload, "RAW TEST");
    
    // 5. Общая длина пакета
    *len = sizeof(struct ethhdr) + sizeof(struct iphdr) + 
           sizeof(struct udphdr) + strlen(payload) + 1;
}

// Прием пакетов с конкретного интерфейса
void sniff_interface(const char *iface_name) {
    int sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (sock < 0) {
        perror("socket");
        return;
    }
    
    // Привязываемся к конкретному интерфейсу
    int ifindex = if_nametoindex(iface_name);
    struct sockaddr_ll sll = {
        .sll_family = AF_PACKET,
        .sll_ifindex = ifindex,
        .sll_protocol = htons(ETH_P_ALL),
    };
    
    if (bind(sock, (struct sockaddr*)&sll, sizeof(sll)) < 0) {
        perror("bind");
        close(sock);
        return;
    }
    
    printf("Сниффинг интерфейса %s...\n", iface_name);
    
    unsigned char buffer[BUFFER_SIZE];
    while (1) {
        ssize_t len = recv(sock, buffer, BUFFER_SIZE, 0);
        if (len < 0) {
            perror("recv");
            break;
        }
        
        // Анализируем пакет
        struct ethhdr *eth = (struct ethhdr *)buffer;
        printf("Получено %zd байт | Протокол: 0x%04x\n", 
               len, ntohs(eth->h_proto));
        
        // Можно анализировать дальше: IP, TCP/UDP и т.д.
    }
    
    close(sock);
}

int main() {
    // Требуются права root!
    
    // Пример 1: Отправка кастомного пакета через eth0
    unsigned char packet[1500];
    size_t packet_len;
    
    create_custom_udp_packet(packet, &packet_len);
    send_raw_packet_via_interface("eth0", packet, packet_len);
    
    // Пример 2: Сниффинг интерфейса (в отдельном потоке)
    // sniff_interface("eth0");
    
    return 0;
}
```



---

## Пример как можно отправить сырой пакет через определенный интерфейс

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <signal.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <linux/if_packet.h>
#include <linux/if_ether.h>
#include <linux/if.h>
#include <arpa/inet.h>
#include <bpf/libbpf.h>
#include <bpf/bpf.h>
#include <sys/ioctl.h>
#include <net/if.h>

// Функция для отправки пакета через конкретный интерфейс в ядерный стек
int inject_to_kernel_stack(int ifindex, const unsigned char *packet, size_t len) {
    // Создаем PACKET socket
    int sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (sock < 0) {
        perror("socket");
        return -1;
    }
    
    // Подготавливаем адрес для отправки
    struct sockaddr_ll sll = {
        .sll_family = AF_PACKET,
        .sll_ifindex = ifindex,      // Конкретный интерфейс!
        .sll_halen = ETH_ALEN,
        .sll_protocol = htons(ETH_P_ALL),
    };
    
    // Копируем MAC назначения из пакета
    if (len >= ETH_ALEN) {
        memcpy(sll.sll_addr, packet, ETH_ALEN);
    }
    
    // ВАЖНО: Отправляем пакет с флагом, чтобы он попал в ядерный стек
    // MSG_PROXY имитирует, что пакет пришел снаружи
    int flags = 0;
    
    // Для Linux можно использовать sockaddr_ll с правильными параметрами
    // Ядро увидит этот пакет как пришедший "извне"
    
    // Отправляем пакет
    ssize_t sent = sendto(sock, packet, len, flags, 
                         (struct sockaddr*)&sll, sizeof(sll));
    
    if (sent < 0) {
        perror("sendto");
        close(sock);
        return -1;
    }
    
    printf("Инжектировано %zd байт в интерфейс ifindex=%d\n", sent, ifindex);
    close(sock);
    return 0;
}
```

---
## **Безопасность и предостережения:**

1. **RAW-сокеты могут быть использованы для атак:**
   - IP spoofing (подделка IP)
   - SYN flood
   - Smurf attacks

2. **Ядро Linux ограничивает RAW-сокеты:**
   ```bash
   # Нельзя отправлять пакеты с чужим source IP без дополнительных настроек
   sysctl net.ipv4.conf.eth0.rp_filter=0  # Отключить проверку source IP
   ```

## **Советы для работы с RAW-сокетами:**

1. **Используйте библиотеки:**
   ```c
   // Вместо ручного формирования пакетов
   #include <libnet.h>     // libnet - создание пакетов
   #include <pcap.h>       // libpcap - захват пакетов
   ```

2. **Оптимизация производительности:**
   ```c
   // Используйте буферизацию
   setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &bufsize, sizeof(bufsize));
   
   // Используйте memory-mapped пакеты (PACKET_MMAP)
   // для высокой производительности
   ```

3. **Кросс-платформенность:**
   - Linux: `AF_PACKET`
   - BSD/macOS: `AF_LINK` (BPF)
   - Windows: `SOCK_RAW` (Winsock)
