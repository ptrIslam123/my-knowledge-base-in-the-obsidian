### **1. Проверка поддержки виртуализации**
```bash
egrep -c '(vmx|svm)' /proc/cpuinfo
```
- **Если `≥1`** – процессор поддерживает виртуализацию (Intel VT-x / AMD-V).  
- **Если `0`** – проверьте BIOS/UEFI:  
  - Intel: включите **VT-x** (может называться **Intel Virtualization Technology**).  
  - AMD: включите **AMD-V** (обычно уже активировано).  

---

### **2. Установка KVM/QEMU (дополненная версия)**
```bash
sudo apt update
sudo apt install -y qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virt-manager install virt-viewer libosinfo-bin
```

**Добавьте пользователя в группы (обязательно!):**
```bash
sudo usermod -aG libvirt $(whoami)
sudo usermod -aG kvm $(whoami)
```

**Перезагрузитесь или выполните:**
```bash
newgrp libvirt
```

**Проверка работы libvirt:**
```bash
sudo systemctl enable --now libvirtd
sudo systemctl status libvirtd
```

---

### **3. Настройка сети

KVM/QEMU **может** запускать виртуальные машины (ВМ) без дополнительной настройки сети, но тогда ВМ будут изолированы от интернета и локальной сети. 

## **1. Какие режимы сети есть в KVM/QEMU?**
### **(а) Режим NAT (по умолчанию)**
- **Как работает:**  
  - ВМ получает IP из внутренней подсети (например, `192.168.122.0/24`).  
  - Хост выступает как роутер (маскарадинг, как в домашнем Wi-Fi).  
  - ВМ может выходить в интернет, но недоступна извне.  

- **Плюсы:**  
  - Не требует настройки на хосте.  
  - Безопасен (ВМ скрыта за NAT).  

- **Минусы:**  
  - Нельзя подключиться к ВМ из локальной сети.  

- **Как включить:**  
  ```bash
  sudo virsh net-start default
  sudo virsh net-autostart default  # автостарт при загрузке
  ```

---

### **(б) Режим моста (Bridge)**
- **Как работает:**  
  - ВМ получает **реальный IP из вашей локальной сети** (как физическое устройство).  
  - Хост и ВМ находятся в одной сети (например, `192.168.1.0/24`).  

- **Плюсы:**  
  - ВМ видна в локальной сети (можно сделать сервер).  
  - Нет лишнего NAT, меньше задержек.  

- **Минусы:**  
  - Требует настройки моста на хосте.  
  - Может конфликтовать с DHCP (если IP выдается автоматически).  

- **Как настроить:**  
  ```yaml
  # /etc/netplan/01-netcfg.yaml
  network:
    version: 2
    renderer: networkd
    ethernets:
      enp3s0:  # ваш основной интерфейс (ip a)
        dhcp4: no
    bridges:
      br0:
        interfaces: [enp3s0]
        dhcp4: yes  # или статический IP
        # addresses: [192.168.1.100/24]
        # gateway4: 192.168.1.1
  ```
  Применить:
  ```bash
  sudo netplan apply
  ```

---

### **(в) Изолированная сеть (Private Network)**
- **Как работает:**  
  - ВМ могут общаться **только между собой** (как в виртуальной лаборатории).  
  - Нет доступа в интернет и локальную сеть.  

- **Плюсы:**  
  - Полная изоляция (безопасность).  
  - Хост может общаться с ВМ через виртуальный интерфейс.  

- **Минусы:**  
  - Нужен VPN или проброс портов для доступа извне.  

- **Как создать:**  
  ```bash
  sudo virsh net-define /usr/share/libvirt/networks/isolated.xml
  sudo virsh net-start isolated
  ```

---

### **4. Создание виртуальной машины
#### **Создание ВМ через virt-install**
Базовый синтаксис команды:
```bash
sudo virt-install \
  --name VM_NAME \
  --ram RAM_SIZE \
  --vcpus CPU_CORES \
  --disk path=/var/lib/libvirt/images/VM_NAME.qcow2,size=DISK_SIZE \
  --os-variant OS_TYPE \
  --network network=NETWORK_TYPE \
  --graphics GRAPHICS_TYPE \
  --cdrom /path/to/iso
```

##### **Основные параметры**
| Параметр | Пример | Описание |
|----------|--------|----------|
| `--name` | `ubuntu-server` | Имя ВМ (должно быть уникальным) |
| `--ram` | `2048` | Объем RAM в МБ |
| `--vcpus` | `2` | Количество виртуальных CPU |
| `--disk` | `path=.../vm.qcow2,size=20` | Путь и размер диска (в ГБ) |

##### **Важные опции**
1. **Тип ОС (`--os-variant`)**
   - Узнать доступные варианты:
     ```bash
     osinfo-query os
     ```
   - Примеры: `ubuntu22.04`, `centos7.0`, `windows10`

2. **Сетевые настройки (`--network`)**
   ```bash
   --network network=default  # NAT
   --network bridge=br0       # Мост
   --network none             # Без сети
   ```

3. **Графический интерфейс (`--graphics`)**
   ```bash
   --graphics spice  # Лучший вариант (поддержка clipboard)
   --graphics vnc    # Альтернатива
   --graphics none   # Только консоль
   ```

4. **Видео и звук**
   ```bash
   --video qxl       # Ускоренная графика
   --sound ac97      # Поддержка аудио
   ```

#### **Примеры**
##### **Создание Ubuntu Server (NAT)**
```bash
sudo virt-install \
  --name ubuntu-server \
  --ram 8192 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/ubuntu.qcow2,size=50 \
  --os-variant ubuntu22.04 \
  --network network=default \
  --graphics vnc \
  --console pty,target_type=serial \
  --cdrom /path/to/ubuntu-22.04-live-server-amd64.iso
```

##### **Создание Windows 10 (мостовая сеть)**
```bash
sudo virt-install \
  --name win10 \
  --ram 8192 \
  --vcpus 4 \
  --disk path=/var/lib/libvirt/images/win10.qcow2,size=50 \
  --os-variant win10 \
  --network bridge=br0 \
  --graphics vnc \
  --video qxl \
  --cdrom /path/to/Win10_21H2_English.iso
```

---

### **5. Управление ВМ (дополненное)**
### **Основные команды `virsh`**

| Команда                       | Описание                        |
| ----------------------------- | ------------------------------- |
| virsh list --all              | Список всех ВМ                  |
| virsh start myvm              | Запуск ВМ                       |
| virsh shutdown myvm           | Корректное выключение           |
| virsh destroy myvm            | Аварийное выключение            |
| virsh reboot myvm             | Перезагрузка                    |
| virsh suspend myvm            | Приостановка                    |
| virsh resume myvm             | Возобновление                   |
| virsh edit myvm               | Редактирование XML-конфига      |
| virsh dumpxml myvm > myvm.xml | Экспорт конфига                 |
| virsh console myvm            | Подключение к текстовой консоли |
| virsh dominfo myvm            | Информация о ВМ                 |
| virsh net-list --all          | Список сетей                    |
**`myvm`  — это именно значение `--name` из `virt-install`**

---

### Старт и подключение к ВМ

```bash
# Запускает виртуальную машину в фоновом режиме, 
# но не открывает графический интерфейс автоматически
virsh start myvm

# 1) Подключается к гипервизору (`qemu:///system`)
# 2) Находит активную ВМ (`myvm`).
# 3) Проверяет, какой графический протокол используется (SPICE/VNC).
# 4) Открывает графическое окно для взаимодействия с ВМ.
virt-viewer --connect qemu:///system myvm
```
