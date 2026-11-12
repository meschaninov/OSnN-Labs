# Лабораторная работа № 3. Управление пакетами, системными сервисами и настройка сети

Студент: Мещанинов Артём Александрович
Группа: ИС-25

## Цель работы

Настроить репозитории и установить пакеты, научиться управлять сервисами через `systemctl` и `cron`, настроить статический IP-адрес и изучить команды диагностики сети.

Все команды выполнялись от имени root.

---

## Задание 1. Управление пакетами

### Настройка sources.list

В файл `/etc/apt/sources.list` добавлены официальные репозитории Debian и зеркало Яндекса:

```
# Официальные репозитории Debian
deb http://deb.debian.org/debian trixie main contrib non-free non-free-firmware
deb http://deb.debian.org/debian trixie-updates main contrib non-free non-free-firmware
deb http://security.debian.org/debian-security trixie-security main contrib non-free non-free-firmware

# Зеркало (Россия)
deb http://mirror.yandex.ru/debian trixie main contrib non-free non-free-firmware
```

`trixie` — название версии Debian 13. Зеркало — копия репозитория, которая находится ближе и быстрее скачивается.

![sources.list](images/01-sources-list.png)

```bash
apt update
```

Списки пакетов загружены и с официального сервера, и с зеркала Яндекса.

![apt update](images/02-apt-update.png)

### Установка пакетов

```bash
apt install -y vim curl net-tools apache2
```

![Установка пакетов](images/03-apt-install.png)

### Скрипт установки пакетов

```bash
#!/bin/bash
# Использование: sudo ./install_packages.sh
PACKAGES="vim curl net-tools apache2"

if [ "$EUID" -ne 0 ]; then echo "Запустите через sudo"; exit 1; fi

apt-get update
for pkg in $PACKAGES; do
    if dpkg -s "$pkg" &>/dev/null; then
        echo "$pkg уже установлен"
    else
        apt-get install -y "$pkg" && echo "$pkg установлен" || echo "ОШИБКА: $pkg"
    fi
done
```

Скрипт по очереди проверяет каждый пакет (`dpkg -s`) и устанавливает, если его нет. Для проверки сначала был удалён vim — скрипт его установил, а остальные пропустил.

![Скрипт установки](images/04-install-script.png)

Apache работает — страница открывается в браузере на Mac по адресу `http://127.0.0.1:8080`:

![Страница Apache](images/05-apache-browser.png)

---

## Задание 2. Работа с системными сервисами

```bash
systemctl enable apache2 ssh       # включить автозапуск
systemctl is-enabled apache2 ssh   # проверить автозапуск
systemctl stop apache2             # остановить
systemctl start apache2            # запустить
systemctl disable apache2          # выключить автозапуск
systemctl enable --now apache2     # включить автозапуск и запустить
systemctl status apache2           # статус
```

![systemctl](images/06-systemctl.png)

### Перезапуск Apache через cron

```bash
crontab -e
```

Добавлена строка — перезапуск Apache каждый день в 4:00:

```
0 4 * * * /usr/bin/systemctl restart apache2
```

Поля: минута, час, день месяца, месяц, день недели.

Для проверки временно было поставлено `*/2 * * * *` (каждые 2 минуты): по статусу видно, что Apache перезапустился в 00:10:01. Потом расписание вернули на ежедневное.

![cron](images/07-cron.png)

---

## Задание 3. Конфигурация сети

### До настройки

Адрес выдавался автоматически по DHCP (в выводе есть слово `dynamic`):

```bash
cat /etc/network/interfaces
ip a show enp0s8
ip route
```

![Сеть до настройки](images/08-network-before.png)

### Статический IP

В файле `/etc/network/interfaces` задан статический адрес:

```
auto enp0s8
iface enp0s8 inet static
    address 10.0.2.15/24
    gateway 10.0.2.2
```

`address` — IP-адрес, `gateway` — шлюз. После этого система перезагружена.

![interfaces](images/09-interfaces.png)

### Проверка

```bash
ip a show enp0s8
ifconfig enp0s8
ip route
getent hosts debian.org
curl -sI http://deb.debian.org | head -1
```

Слова `dynamic` больше нет — адрес статический. Интернет работает: сайт отвечает `200 OK`. Вместо `ping` используется `curl`, потому что VirtualBox на Mac не пропускает ping наружу.

![Проверка сети](images/10-network-check.png)

---

## Задание 4. Диагностика сетевых проблем

| Команда      | Что делает                                            | Пример               |
| ------------ | ----------------------------------------------------- | -------------------- |
| `traceroute` | показывает путь пакета до сервера                     | `traceroute 8.8.8.8` |
| `netstat`    | показывает открытые порты и маршруты (старая команда) | `netstat -tulpn`     |
| `ss`         | то же, что netstat, но новее и быстрее                | `ss -tulpn`          |

### traceroute

```bash
traceroute -n 8.8.8.8
traceroute -T -p 443 ya.ru
```

Первый узел — шлюз VirtualBox `10.0.2.2`, дальше `* * *`: VirtualBox не показывает промежуточные узлы.

![traceroute](images/11-traceroute.png)

### netstat

```bash
netstat -tulpn   # открытые порты
netstat -rn      # таблица маршрутов
```

Порт 22 слушает `sshd` (SSH), порт 80 — `apache2`. Маршрут по умолчанию идёт через шлюз `10.0.2.2`.

![netstat](images/12-netstat.png)

### ss

```bash
ss -tulpn                  # открытые порты
ss -tn state established   # активные соединения
ss -s                      # общая статистика
```

Видны те же порты 22 и 80 и текущее SSH-подключение с Mac.

![ss](images/13-ss.png)

---

## Вывод

Настроены репозитории, установлены пакеты, в том числе скриптом. Настроен автозапуск Apache и SSH и ежедневный перезапуск Apache через cron. Настроен статический IP-адрес. Изучены команды `traceroute`, `netstat` и `ss`.
