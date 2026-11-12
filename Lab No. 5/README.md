# Лабораторная работа № 5. Комплексная работа: сеть из двух ВМ, веб-сервер, NFS, Samba, мониторинг

ФИО студента: Мещанинов Артём Александрович
Группа: ИС-25

## Цель работы

Объединить две виртуальные машины Debian в общую сеть, развернуть на одной из них веб-сервер nginx, сетевые файловые ресурсы NFS и Samba и проверить их доступность со второй машины; проанализировать сетевой трафик с помощью tcpdump и нагрузку на систему с помощью top.

## Используемые средства

| Компонент    | Версия                                                       |
| ------------ | ------------------------------------------------------------ |
| Хост-система | macOS, Apple M4                                              |
| Гипервизор   | Oracle VirtualBox 7.2                                        |
| Гостевые ОС  | Debian GNU/Linux 13 «Trixie», arm64 — 2 виртуальные машины   |
| Сервисы      | nginx, nfs-kernel-server, samba, tcpdump, apache2-utils (ab) |

## Схема стенда

| ВМ     | Hostname        | Адаптер 1 — NAT (`enp0s8`)      | Адаптер 2 — внутренняя сеть `intnet` (`enp0s9`) | Роль                            |
| ------ | --------------- | ------------------------------- | ----------------------------------------------- | ------------------------------- |
| Сервер | `debian-server` | 10.0.2.15, SSH с Mac: порт 2222 | 192.168.50.10/24                                | nginx, NFS, Samba, tcpdump      |
| Клиент | `debian-client` | 10.0.2.15, SSH с Mac: порт 2223 | 192.168.50.20/24                                | curl, NFS-клиент, smbclient, ab |

Адаптер NAT даёт каждой машине выход в интернет и SSH-доступ с Mac через проброс портов. Адаптер «Внутренняя сеть» объединяет машины в изолированную сеть `192.168.50.0/24`, через которую они взаимодействуют. Одинаковый адрес 10.0.2.15 на обеих машинах не конфликтует: у каждой ВМ свой изолированный NAT.

---

## Задание 1. Настройка сети для двух виртуальных машин

### 1.1. Создание второй ВМ

Вторая ВМ создана полным клонированием первой с генерацией новых MAC-адресов для всех адаптеров. Чтобы машины не конфликтовали, у клона изменены имя компьютера, machine-id и ключи SSH-сервера:

```bash
# КЛИЕНТ
sudo hostnamectl set-hostname debian-client
sudo sed -i 's/\bdebian\b/debian-client/' /etc/hosts
sudo rm /etc/machine-id && sudo systemd-machine-id-setup
sudo rm /etc/ssh/ssh_host_* && sudo dpkg-reconfigure openssh-server

# СЕРВЕР
sudo hostnamectl set-hostname debian-server
sudo sed -i 's/\bdebian\b/debian-server/' /etc/hosts
```

Правило проброса SSH для клиента изменено на порт хоста **2223**, так как две ВМ не могут занять один порт Mac.

![Клонирование ВМ](images/01-clone-vm.png)

### 1.2. Внутренняя сеть

Для обеих ВМ включён адаптер 2 типа **«Внутренняя сеть»** с именем `intnet`. Во внутренней сети нет DHCP, поэтому адреса заданы статически в `/etc/network/interfaces`:

```
# сервер                          # клиент
auto enp0s9                       auto enp0s9
iface enp0s9 inet static          iface enp0s9 inet static
    address 192.168.50.10/24          address 192.168.50.20/24
```

Шлюз для этого интерфейса не указан: выход в интернет идёт через NAT-адаптер.

![Адаптер 2 — внутренняя сеть](images/02-intnet-adapter.png)

```bash
sudo systemctl restart networking
ip -br a
```

![Адреса интерфейсов на обеих ВМ](images/03-ip-both.png)

### 1.3. Проверка доступности

```bash
# СЕРВЕР
ping -c 3 192.168.50.20
# КЛИЕНТ
ping -c 3 192.168.50.10
```

![ping между машинами](images/04-ping.png)

SSH-сервер установлен на обеих машинах (копия первой ВМ). Подключение с клиента на сервер — сначала по паролю, затем настроен вход по ключу:

```bash
# КЛИЕНТ
ssh user@192.168.50.10          # по паролю; hostname → debian-server
ssh-keygen -t ed25519
ssh-copy-id user@192.168.50.10
ssh user@192.168.50.10          # без пароля
```

![SSH с клиента на сервер](images/05-ssh-client-server.png)

### 1.4. Файрвол

На обеих машинах работает UFW (настроен в лабораторной работе № 4): входящие запрещены, кроме явно разрешённых; SSH разрешён. Ping (ICMP echo-request) UFW пропускает по умолчанию — правило находится в `/etc/ufw/before.rules` и применяется раньше пользовательских правил.

```bash
sudo ufw status verbose
sudo grep -n echo-request /etc/ufw/before.rules
```

![Правила UFW: SSH и ping разрешены](images/06-ufw.png)

---

## Задание 2. Настройка веб-сервера и клиента

### 2.1. Установка nginx

Порт 80 был занят Apache из лабораторной работы № 3, поэтому Apache остановлен, исключён из автозапуска, а задание cron с его перезапуском удалено:

```bash
# СЕРВЕР
sudo systemctl disable --now apache2
sudo crontab -l | grep -v apache2 | sudo crontab -

sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
sudo ss -tlnp | grep :80
```

![Установка nginx](images/07-nginx-install.png)

### 2.2. HTML-страница

```bash
sudo tee /var/www/html/index.html > /dev/null << 'EOF'
<html>
<head><title>Test Page</title></head>
<body><h1>Hello from Debian!</h1></body>
</html>
EOF
curl http://localhost
```

![Создание страницы](images/08-index-html.png)

### 2.3. Проверка с другой ВМ

```bash
# КЛИЕНТ
curl http://192.168.50.10
curl -I http://192.168.50.10      # HTTP/1.1 200 OK, Server: nginx
```

![curl с клиента](images/09-curl-ip.png)

### 2.4. Доступ по доменному имени через /etc/hosts

Файл `/etc/hosts` сопоставляет имена с IP-адресами локально и проверяется раньше DNS:

```bash
# КЛИЕНТ
echo "192.168.50.10  debian-server  lab.local" | sudo tee -a /etc/hosts
getent hosts lab.local
curl http://lab.local
```

![Доступ по имени lab.local](images/10-curl-hosts.png)

---

## Задание 3. Настройка NFS-сервера

### 3.1–3.4. Сервер

```bash
# СЕРВЕР
sudo apt install -y nfs-kernel-server
sudo mkdir -p /srv/nfs/share
sudo chown nobody:nogroup /srv/nfs/share
sudo chmod 777 /srv/nfs/share

echo "/srv/nfs/share 192.168.50.20(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -ra                       # перечитать /etc/exports
sudo systemctl restart nfs-kernel-server
sudo exportfs -v
sudo ufw allow from 192.168.50.20 to any port 2049 proto tcp
```

Опции экспорта: `rw` — чтение и запись; `sync` — сервер подтверждает запись только после сохранения на диск; `no_subtree_check` — отключение проверки поддерева. Доступ разрешён только адресу клиента.

![Настройка NFS-сервера](images/11-nfs-server.png)

### 3.5–3.6. Клиент и проверка

```bash
# КЛИЕНТ
sudo apt install -y nfs-common
sudo mkdir -p /mnt/nfs/share
sudo mount 192.168.50.10:/srv/nfs/share /mnt/nfs/share
df -h -t nfs4
echo "привет с клиента" > /mnt/nfs/share/from_client.txt
```

```bash
# СЕРВЕР
ls -ln /srv/nfs/share
cat /srv/nfs/share/from_client.txt
```

Файл, созданный на клиенте, появился на сервере.

![Файл с клиента виден на сервере](images/12-nfs-check.png)

### 3.7. Постоянное монтирование через /etc/fstab

```bash
# КЛИЕНТ
echo "192.168.50.10:/srv/nfs/share  /mnt/nfs/share  nfs  defaults,_netdev,nofail  0  0" | sudo tee -a /etc/fstab
sudo umount /mnt/nfs/share
sudo systemctl daemon-reload
sudo mount -a
findmnt /mnt/nfs/share
```

`_netdev` — монтировать после поднятия сети; `nofail` — не прерывать загрузку, если сервер недоступен. После перезагрузки клиента ресурс смонтирован автоматически.

![fstab и монтирование после перезагрузки](images/13-nfs-fstab.png)

### 3.8. Права доступа через UID/GID

NFS проверяет права по числовым UID/GID, а не по именам пользователей, поэтому у пользователя должны совпадать идентификаторы на обеих машинах. Созданы пользователи с одинаковыми UID/GID на сервере и клиенте:

```bash
# НА ОБЕИХ МАШИНАХ
sudo groupadd -g 3000 nfsgroup
sudo useradd -m -u 3001 -g 3000 -s /bin/bash nfsuser1
sudo useradd -m -u 3002 -g 3000 -s /bin/bash nfsuser2

# СЕРВЕР
sudo mkdir /srv/nfs/share/private1 /srv/nfs/share/team
sudo chown 3001:3000 /srv/nfs/share/private1 && sudo chmod 700 /srv/nfs/share/private1
sudo chown root:3000 /srv/nfs/share/team     && sudo chmod 2770 /srv/nfs/share/team

# КЛИЕНТ
sudo -u nfsuser1 touch /mnt/nfs/share/private1/a     # успех — владелец каталога
sudo -u nfsuser2 touch /mnt/nfs/share/private1/b     # Permission denied
sudo -u nfsuser2 touch /mnt/nfs/share/team/c         # успех — общая группа
ls -ln /mnt/nfs/share/private1 /mnt/nfs/share/team
```

Пользователь root клиента на сервере по умолчанию отображается в `nobody` (опция `root_squash`), чтобы root любого клиента не получил права root на сервере.

![Права через UID/GID](images/14-nfs-uid.png)

---

## Задание 4. Настройка Samba-сервера

### 4.1–4.4. Установка и настройка

```bash
# СЕРВЕР
sudo apt install -y samba
sudo smbpasswd -a admin
sudo smbpasswd -a user1
sudo pdbedit -L

sudo mkdir /home/admin/shared
sudo chown admin:developers /home/admin/shared
sudo chmod 2770 /home/admin/shared
```

Samba хранит собственную базу паролей (формат NT-хешей), поэтому пользователям заданы отдельные SMB-пароли. Каталог отдан группе `developers`, в которую входят `admin` и `user1`: Samba применяет обычные права Linux, и при владельце `admin:admin` пользователь `user1` смог бы только читать.

Секция в конце `/etc/samba/smb.conf`:

```ini
[shared]
   path = /home/admin/shared
   valid users = admin, user1
   read only = no
   create mask = 0660
   directory mask = 2770
   hosts allow = 192.168.50.0/24 10.0.2.0/24 127.0.0.1
   hosts deny = ALL
```

`hosts allow` / `hosts deny` ограничивают доступ по IP-адресам: внутренняя сеть, NAT (для подключения с Mac) и локальный адрес.

```bash
testparm -s
sudo systemctl restart smbd
sudo ufw allow from 192.168.50.0/24 to any app Samba
sudo ufw allow from 10.0.2.0/24 to any port 445 proto tcp
```

![Настройка Samba](images/15-samba-config.png)

### 4.5. Подключение с клиента

```bash
# КЛИЕНТ
sudo apt install -y smbclient
smbclient -L //192.168.50.10 -U admin
smbclient //192.168.50.10/shared -U admin
#   smb: \> put /etc/hostname host.txt
#   smb: \> ls
#   smb: \> exit
```

![Подключение через smbclient](images/16-smbclient.png)

На сервере загруженный файл появился в `/home/admin/shared`, активные подключения видны в `sudo smbstatus`.

![Файл на сервере и smbstatus](images/17-samba-server-check.png)

### 4.6. Ограничение по IP

Для проверки в `hosts allow` временно указан адрес `192.168.50.99` вместо подсети — клиент `192.168.50.20` получил отказ в доступе. Затем настройка возвращена.

![Отказ в доступе по IP](images/18-samba-deny.png)

### 4.7. Проверка через GUI

Для подключения с Mac в VirtualBox добавлен проброс порта сервера 4450 → 445. В Finder: «Подключение к серверу» (Cmd+K) → `smb://127.0.0.1:4450/shared`, пользователь `admin`. Файл, перетащенный в окно Finder, появился в `/home/admin/shared` на сервере.

![Ресурс Samba в Finder](images/19-samba-finder.png)

---

## Задание 5. Мониторинг сетевой активности

Трафик между ВМ проходит через интерфейс внутренней сети `enp0s9`, поэтому захват выполнялся на нём (на `enp0s8` виден только трафик NAT — интернет и SSH с Mac).

```bash
# СЕРВЕР
sudo apt install -y tcpdump
sudo tcpdump -D
```

| Что анализировалось | Команда на сервере                     | Действие на клиенте                                 |
| ------------------- | -------------------------------------- | --------------------------------------------------- |
| ping                | `sudo tcpdump -i enp0s9 -n icmp`       | `ping -c 3 192.168.50.10`                           |
| HTTP-запрос к nginx | `sudo tcpdump -i enp0s9 -n -A port 80` | `curl http://192.168.50.10`                         |
| NFS                 | `sudo tcpdump -i enp0s9 -n port 2049`  | `ls /mnt/nfs/share`                                 |
| Samba               | `sudo tcpdump -i enp0s9 -n port 445`   | `smbclient //192.168.50.10/shared -U admin -c 'ls'` |

![tcpdump: ICMP](images/20-tcpdump-icmp.png)

Разбор HTTP-запроса: первые три пакета — тройное рукопожатие TCP (флаги `[S]` — SYN от клиента, `[S.]` — SYN-ACK от сервера, `[.]` — ACK от клиента), затем пакеты с флагом `[P.]` — запрос `GET / HTTP/1.1` и ответ `HTTP/1.1 200 OK` с HTML-страницей, в конце `[F.]` — закрытие соединения. С флагом `-A` содержимое пакетов видно открытым текстом, так как HTTP не шифруется. Клиент использует случайный порт, сервер — порт 80.

![tcpdump: HTTP-запрос к nginx](images/21-tcpdump-http.png)

![tcpdump: NFS](images/22-tcpdump-nfs.png)

Сохранение захвата в файл и чтение:

```bash
sudo tcpdump -i enp0s9 -w traffic.pcap      # на клиенте — ping, curl, обращение к NFS; затем Ctrl+C
sudo tcpdump -r traffic.pcap -n | head -30
```

![Сохранение захвата в traffic.pcap](images/23-tcpdump-pcap.png)

Файл скопирован на Mac (`scp -P 2222 user@127.0.0.1:traffic.pcap ~/Desktop/`) и открыт в Wireshark.

![traffic.pcap в Wireshark](images/24-wireshark.png)

---

## Задание 6. Использование top для мониторинга системы

```bash
top
```

| Строка                                                       | Что показывает                                                                                                                                                                        |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `top - … up …, load average: 0.35, 0.20, 0.10`               | время работы и **load average** — среднее число процессов, готовых к выполнению, за 1, 5 и 15 минут; сравнивается с числом ядер (у ВМ 2 ядра — значение 2.0 означает полную загрузку) |
| `Tasks: … total, … running, … sleeping, … stopped, … zombie` | количество процессов по состояниям, в том числе **зомби**                                                                                                                             |
| `%Cpu(s): us, sy, ni, id, wa, hi, si, st`                    | **использование процессора**: `us` — пользовательские программы, `sy` — ядро, `id` — простой, `wa` — ожидание ввода-вывода, `st` — время, отобранное гипервизором                     |
| `MiB Mem` / `MiB Swap`                                       | **использование памяти**: всего, свободно, занято, кэш; `avail Mem` — доступно программам                                                                                             |

Колонки таблицы процессов: `PID`, `USER`, `RES` (физическая память), `S` (состояние: R — работает, S — спит, Z — зомби), `%CPU`, `%MEM`, `COMMAND`. Клавиши: `P` — сортировка по CPU, `M` — по памяти, `1` — по ядрам, `q` — выход.

![top в обычном режиме](images/25-top-overview.png)

**Зомби-процесс** — процесс, который уже завершился, но родитель не забрал его код возврата (не вызвал `wait`), поэтому запись осталась в таблице процессов. Ресурсов он не потребляет и исчезает, когда завершается родитель. Для демонстрации создан зомби на 60 секунд:

```bash
python3 -c "import os,time; os.fork()==0 or time.sleep(60)" &
ps -eo pid,ppid,stat,cmd | grep -w Z
```

![Зомби-процесс в top](images/26-top-zombie.png)

### Nginx под нагрузкой

Отображение только процессов nginx с обновлением раз в секунду:

```bash
# СЕРВЕР
top -d 1 -p $(pgrep -d, nginx)
```

У nginx один главный процесс (master, от root — нужен для открытия порта 80) и рабочие процессы (worker, от `www-data`) по числу ядер. В простое worker'ы почти не используют процессор, потребление памяти — несколько мегабайт.

![nginx в top без нагрузки](images/27-top-nginx-idle.png)

Нагрузка создана с клиента утилитой Apache Benchmark (пакет `apache2-utils`):

```bash
# КЛИЕНТ
sudo apt install -y apache2-utils
ab -n 1000 -c 10 http://192.168.50.10/
ab -n 200000 -c 100 http://192.168.50.10/
```

`-n` — общее число запросов, `-c` — число одновременных запросов.

![nginx в top под нагрузкой](images/28-top-nginx-load.png)

Во время нагрузки `%CPU` worker-процессов nginx вырос, увеличился load average, в строке `%Cpu(s)` уменьшилось время простоя (`id`) и выросли `sy`/`si` — обработка сетевых пакетов в ядре. Потребление памяти nginx практически не изменилось: отдача статической страницы памяти не требует.

![Результат ab](images/29-ab-result.png)

По итогам `ab`: `Requests per second` — производительность сервера, `Time per request` — время обработки запроса, `Failed requests` — 0.

---

## Вывод

Две виртуальные машины Debian объединены во внутреннюю сеть VirtualBox `192.168.50.0/24` со статическими адресами; проверена связь командой `ping`, настроен SSH-доступ между машинами по паролю и по ключу; UFW защищает машины, пропуская SSH и ping. На сервере развёрнут nginx со своей HTML-страницей, доступной с клиента по IP-адресу и по имени из `/etc/hosts`. Настроен NFS-сервер с экспортом `/srv/nfs/share` для клиента, постоянным монтированием через `/etc/fstab` и разграничением прав через совпадающие UID/GID. Настроен Samba-ресурс для пользователей `admin` и `user1` с ограничением доступа по IP; подключение проверено через `smbclient` и Finder на Mac. С помощью tcpdump проанализирован трафик ICMP, HTTP, NFS и SMB, захват сохранён в файл и открыт в Wireshark. С помощью top изучены показатели загрузки процессора, памяти, load average и зомби-процессы, а также потребление ресурсов nginx под нагрузкой, созданной утилитой ab.
