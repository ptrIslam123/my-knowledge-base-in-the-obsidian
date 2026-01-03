## Проверка системных требований

### Проверка версии Ubuntu и ядра
```bash
# Проверка версии Ubuntu
lsb_release -a

# Проверка версии ядра
uname -r

# Рекомендуется: Ubuntu 20.04+ с ядром 5.4+
```

### Проверка поддержки IOMMU
```bash
# Для Intel процессоров
grep -E "svm|vmx" /proc/cpuinfo

# Проверка включения IOMMU в ядре
cat /proc/cmdline | grep -i iommu

# Проверка наличия IOMMU групп
ls /sys/kernel/iommu_groups/ | wc -l
```

### Определение сетевых интерфейсов
```bash
# Список всех сетевых интерфейсов
ip link show

# Детальная информация о PCI устройствах
lspci | grep -i ethernet

# Подробная информация о конкретном устройстве
lspci -v -s 0000:00:03.0  # замените на свой PCI адрес
```

---

## Установка DPDK

### Установка из репозитория Ubuntu (простой способ)
```bash
# Обновление списка пакетов
sudo apt update

# Установка DPDK и инструментов
sudo apt install -y dpdk dpdk-dev libdpdk-dev dpdk-igb-uio-dkms

# Установка дополнительных утилит
sudo apt install -y hugepages python3-pciutils
```

### Установка из исходников (рекомендуется для последней версии)
```bash
# Установка зависимостей для сборки
sudo apt install -y build-essential meson ninja-build python3-pip \
    python3-pyelftools libnuma-dev pkg-config

# Скачивание последней стабильной версии
cd /opt
sudo wget http://fast.dpdk.org/rel/dpdk-23.11.tar.xz
sudo tar -xf dpdk-23.11.tar.xz
cd dpdk-23.11

# Сборка и установка
meson setup build
cd build
ninja
sudo ninja install
sudo ldconfig

# Настройка переменных окружения
echo 'export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Проверка установки
```bash
# Проверка версии DPDK
pkg-config --modversion libdpdk

# Поиск утилит DPDK
find /usr -name "dpdk-devbind.py" 2>/dev/null
find /usr -name "testpmd" 2>/dev/null
```

---

## Настройка ядра и модулей

### Включение IOMMU (для работы VFIO)
```bash
# Редактирование конфигурации GRUB
sudo nano /etc/default/grub
```

Найдите строку `GRUB_CMDLINE_LINUX_DEFAULT` и добавьте параметры intel/amd_iommu=on:
```
# Для Intel процессоров
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash intel_iommu=on iommu=pt"

# Для AMD процессоров
GRUB_CMDLINE_LINUX_DEFAULT="quiet splash amd_iommu=on iommu=pt"
```

```bash
# Обновление GRUB
sudo update-grub
```

### Настройка модулей ядра
```bash
# Создание файла конфигурации модулей
sudo nano /etc/modules-load.d/dpdk.conf
```

Добавьте строки:
```
uio
vfio
vfio_iommu_type1
vfio-pci
vfio_virqfd
```

### Настройка параметров модулей
```bash
# Создание файла параметров модулей
sudo nano /etc/modprobe.d/vfio.conf
```

Добавьте (если IOMMU не работает или для виртуальных машин):
```
options vfio enable_unsafe_noiommu_mode=1
options vfio-pci disable_idle_d3=1
```

### Принудительная загрузка модулей
```bash
# Загрузка модулей сейчас
sudo modprobe uio
sudo modprobe vfio
sudo modprobe vfio_iommu_type1
sudo modprobe vfio-pci

# Проверка загрузки
lsmod | grep -E "uio|vfio"
```

---

## Настройка hugepages

Hugepages критически важны для производительности DPDK.

### Временная настройка hugepages
```bash
# Настройка 2MB hugepages (например, 1024 страниц = 2GB)
echo 1024 | sudo tee /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# Настройка 1GB hugepages (если процессор поддерживает)
echo 2 | sudo tee /sys/kernel/mm/hugepages/hugepages-1048576kB/nr_hugepages
```

### Постоянная настройка hugepages
```bash
# Создание файла конфигурации sysctl
sudo nano /etc/sysctl.d/99-hugepages.conf
```

Добавьте:
```
# 2MB hugepages - 1024 страницы = 2GB
vm.nr_hugepages = 1024

# Максимальное количество отображаемых сегментов памяти
vm.max_map_count = 65535

# Минимальная память для hugepages
vm.min_free_kbytes = 1048576
```

### Монтирование hugepages
```bash
# Создание точки монтирования
sudo mkdir -p /mnt/huge

# Монтирование 2MB hugepages
sudo mount -t hugetlbfs nodev /mnt/huge

# Добавление в fstab для автоматического монтирования
echo "nodev /mnt/huge hugetlbfs defaults 0 0" | sudo tee -a /etc/fstab
```

### Проверка настройки hugepages
```bash
# Проверка количества hugepages
cat /proc/meminfo | grep -i huge

# Проверка монтирования
mount | grep huge
```

---

## 5. Привязка сетевых интерфейсов к DPDK

### Определение устройств для привязки
```bash
# Просмотр всех сетевых устройств
sudo dpdk-devbind.py --status

# Запись PCI адресов нужных интерфейсов
# Например: 0000:00:03.0, 0000:00:04.0
```

### Отключение интерфейсов (если они используются)
```bash
# Определение имени интерфейса по PCI адресу
INTERFACE_NAME=$(ls /sys/bus/pci/devices/0000:00:03.0/net/)
echo "Interface: $INTERFACE_NAME"

# Отключение интерфейса
sudo ip link set dev $INTERFACE_NAME down

# Удаление IP адресов
sudo ip addr flush dev $INTERFACE_NAME
```

### Привязка к VFIO драйверу
```bash
# Включение небезопасного режима (если IOMMU отключен)
echo 1 | sudo tee /sys/module/vfio/parameters/enable_unsafe_noiommu_mode

# Привязка через dpdk-devbind.py
sudo dpdk-devbind.py --bind=vfio-pci 0000:00:03.0

# Привязка нескольких устройств
sudo dpdk-devbind.py --bind=vfio-pci 0000:00:03.0 0000:00:04.0
```

### Альтернативный метод: ручная привязка через sysfs
```bash
# Отвязка от текущего драйвера
echo "0000:00:03.0" | sudo tee /sys/bus/pci/devices/0000:00:03.0/driver/unbind

# Установка driver_override
echo "vfio-pci" | sudo tee /sys/bus/pci/devices/0000:00:03.0/driver_override

# Привязка к vfio-pci
echo "0000:00:03.0" | sudo tee /sys/bus/pci/drivers/vfio-pci/bind
```

### Альтернативный драйвер: uio_pci_generic
```bash
# Если vfio-pci не работает, используйте uio_pci_generic
sudo modprobe uio_pci_generic
sudo dpdk-devbind.py --bind=uio_pci_generic 0000:00:03.0
```

### Альтернативный драйвер: igb_uio
```bash
# Установка igb_uio (если доступен)
sudo modprobe igb_uio
sudo dpdk-devbind.py --bind=igb_uio 0000:00:03.0
```

### Проверка привязки
```bash
# Проверка через lspci
lspci -k -s 0000:00:03.0

# Проверка через dpdk-devbind
sudo dpdk-devbind.py --status
```

---

## Настройка персистентности (сохранение после перезагрузки)

### Настройка автоматической привязки через udev
```bash
# Определение vendor и device ID
lspci -n -s 0000:00:03.0
# Вывод должен быть типа: 00:03.0 0200: 8086:100e (rev 02)
# Vendor ID: 8086, Device ID: 100e
```

```bash
# Создание правил udev
sudo nano /etc/udev/rules.d/99-vfio-pci.rules
```

Добавьте для каждого устройства:
```
# Intel 82540EM (замените 8086 и 100e на свои ID)
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="0x100e", ATTR{driver_override}="vfio-pci", RUN+="/bin/sh -c 'echo %k > /sys/bus/pci/drivers/vfio-pci/bind'"

# Можно добавить несколько устройств
ACTION=="add", SUBSYSTEM=="pci", ATTR{vendor}=="0x8086", ATTR{device}=="0x10fb", ATTR{driver_override}="vfio-pci", RUN+="/bin/sh -c 'echo %k > /sys/bus/pci/drivers/vfio-pci/bind'"
```

### Настройка автозагрузки модулей
```bash
sudo nano /etc/modules-load.d/dpdk.conf
```
Убедитесь, что есть строки:
```
vfio
vfio_iommu_type1
vfio-pci
uio
```

### Настройка переменных окружения
```bash
# Добавление в .bashrc
echo 'export RTE_SDK=/usr/local/share/dpdk' >> ~/.bashrc
echo 'export RTE_TARGET=x86_64-native-linux-gcc' >> ~/.bashrc
echo 'export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig:$PKG_CONFIG_PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/lib:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Создание скрипта для быстрой настройки
```bash
# Создание скрипта
sudo nano /usr/local/bin/dpdk-setup.sh
```

```bash
#!/bin/bash
# DPDK setup script

# Загрузка модулей
modprobe vfio-pci 2>/dev/null
modprobe uio_pci_generic 2>/dev/null

# Включение небезопасного режима
echo 1 > /sys/module/vfio/parameters/enable_unsafe_noiommu_mode 2>/dev/null

# Привязка устройств (укажите свои PCI адреса)
dpdk-devbind.py --bind=vfio-pci 0000:00:03.0 2>/dev/null

echo "DPDK setup completed"
```

```bash
# Сделать скрипт исполняемым
sudo chmod +x /usr/local/bin/dpdk-setup.sh
```

---

## Проверка работоспособности

### Базовая проверка
```bash
# Проверка статуса устройств
sudo dpdk-devbind.py --status

# Проверка hugepages
grep Huge /proc/meminfo

# Проверка прав доступа к VFIO
ls -la /dev/vfio/
```

### Проверка прав доступа для пользователя
```bash
# Создание группы vfio
sudo groupadd vfio 2>/dev/null

# Добавление пользователя в группу
sudo usermod -aG vfio $USER

# Настройка прав udev для группы
sudo nano /etc/udev/rules.d/99-vfio-permissions.rules
```

Добавьте:
```
# Дать группе vfio доступ к VFIO
SUBSYSTEM=="vfio", OWNER="root", GROUP="vfio", MODE="0660"
SUBSYSTEM=="vfio", KERNEL=="vfio", OWNER="root", GROUP="vfio", MODE="0666"
```

```bash
# Перезагрузка udev правил
sudo udevadm control --reload-rules
sudo udevadm trigger
```

### Проверка от пользователя
```bash
# Переключение на пользователя (или перелогин)
su - $USER

# Проверка доступа
ls -la /dev/vfio/
dpdk-devbind.py --status 2>/dev/null | grep vfio-pci
```

---

## Запуск тестовых приложений

### 1 TestPMD (базовое тестирование)
```bash
# Запуск TestPMD с одним портом
sudo dpdk-testpmd -l 0-1 -n 4 -- -i \
    --port-topology=chained \
    --nb-cores=1 \
    --total-num-mbufs=2048

# В интерактивном режиме TestPMD:
> show port info all
> start
> show port stats all
> stop
> quit
```

### 2 Запуск с несколькими портами
```bash
# TestPMD с двумя портами
sudo dpdk-testpmd -l 0-2 -n 4 -- -i \
    --port-topology=chained \
    --nb-cores=2 \
    --rxq=1 --txq=1 \
    --forward-mode=io
```

### 3 L2 Forwarding (пример приложения)
```bash
# Сборка примеров (если есть исходники)
cd /usr/local/share/dpdk/examples/l2fwd
make

# Запуск l2fwd
sudo ./build/l2fwd -l 0-1 -n 4 -- -p 0x01
```

### 4 Проверка производительности
```bash
# TestPMD в режиме статистики
sudo dpdk-testpmd -l 0-2 -n 4 -- -i \
    --port-topology=chained \
    --nb-cores=2 \
    --forward-mode=macswap \
    --stats-period=5
```

---
