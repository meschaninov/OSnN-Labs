# Лабораторная работа № 4. Безопасность системы и файловые системы

ФИО студента: Мещанинов Артём Александрович
Группа: ИС-25

## Цель работы

Настроить межсетевой экран UFW, автоматическую установку обновлений безопасности и защиту SSH от подбора паролей с помощью fail2ban; подключить, разметить и автоматически смонтировать новый диск; настроить контроль заполнения дискового пространства.

## Используемые средства

| Компонент    | Версия                              |
| ------------ | ----------------------------------- |
| Хост-система | macOS, Apple M4                     |
| Гипервизор   | Oracle VirtualBox 7.2               |
| Гостевая ОС  | Debian GNU/Linux 13 «Trixie», arm64 |
| Новый диск   | VDI, 5 ГБ, в системе — `/dev/sdb`   |

---

## 1. Настройка файрвола

### 1.1. Установка и настройка UFW

UFW (Uncomplicated Firewall) — упрощённый интерфейс к netfilter: правила UFW преобразуются в правила iptables. Политика — «запрещено всё, что не разрешено». Правило для SSH добавлено **до** включения файрвола, иначе текущая SSH-сессия была бы разорвана.

```bash
sudo apt install -y ufw
sudo ufw default deny incoming     # входящие — запрещены
sudo ufw default allow outgoing    # исходящие — разрешены
sudo ufw allow 22/tcp              # SSH
sudo ufw allow 80/tcp              # HTTP
sudo ufw allow 443/tcp             # HTTPS
sudo ufw enable
```

![Настройка и включение UFW](images/01-ufw-setup.png)

### 1.2. Проверка

```bash
sudo ufw status verbose
sudo ufw status numbered
```

![ufw status](images/02-ufw-status.png)

```bash
sudo iptables -L -n -v | head -40
sudo iptables -S | grep -E "dport (22|80|443)"
```

В выводе iptables видны цепочки `ufw-*`, созданные UFW, и правила `ACCEPT` для портов 22, 80, 443. Ответы на исходящие соединения пропускаются благодаря отслеживанию состояния соединений (`ESTABLISHED,RELATED`).

![Правила в iptables](images/03-iptables.png)

Проверка запрета: на ВМ запущен тестовый веб-сервер на порту 8000 (`python3 -m http.server 8000`), в VirtualBox добавлен проброс 8000 → 8000. С Mac страница `http://127.0.0.1:8000` не открывается — порт не разрешён в UFW, тогда как Apache на 8080 → 80 открывается.

![Проверка блокировки порта 8000](images/04-ufw-block-test.png)

---

## 2. Обеспечение безопасности системы

### 2.1. Автоматические обновления безопасности

```bash
sudo apt install -y unattended-upgrades apt-listchanges
sudo dpkg-reconfigure -plow unattended-upgrades
cat /etc/apt/apt.conf.d/20auto-upgrades
```

Файл `20auto-upgrades` включает ежедневное обновление списков пакетов и автоматическую установку обновлений:

```
APT::Periodic::Update-Package-Lists "1";
APT::Periodic::Unattended-Upgrade "1";
```

Какие обновления устанавливать, задаётся в `/etc/apt/apt.conf.d/50unattended-upgrades` — по умолчанию разрешён источник `Debian-Security`. Запуск выполняют таймеры systemd `apt-daily` и `apt-daily-upgrade`.

```bash
grep -n "Debian-Security" /etc/apt/apt.conf.d/50unattended-upgrades
sudo unattended-upgrade --dry-run --debug 2>&1 | tail -20
systemctl list-timers 'apt-daily*'
```

![Настройка unattended-upgrades](images/05-unattended-upgrades.png)

### 2.2. fail2ban

fail2ban отслеживает журналы, находит неудачные попытки входа и при превышении порога блокирует IP-адрес правилом файрвола на заданное время.

```bash
sudo apt install -y fail2ban python3-systemd
sudo nano /etc/fail2ban/jail.local
```

```ini
[DEFAULT]
bantime  = 10m
findtime = 10m
maxretry = 5
backend  = systemd

[sshd]
enabled = true
port    = ssh
```

- `maxretry = 5` за `findtime = 10m` — 5 неудачных попыток за 10 минут приводят к блокировке;
- `bantime = 10m` — длительность блокировки;
- `backend = systemd` — в Debian 13 журнал `/var/log/auth.log` по умолчанию отсутствует, события читаются из journald;
- настройки записаны в `jail.local`, так как `jail.conf` перезаписывается при обновлении пакета.

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

![Настройка и запуск fail2ban](images/06-fail2ban-setup.png)

Проверка: с Mac выполнено несколько попыток входа с неверным паролем (`ssh -p 2222 hacker@127.0.0.1`). После пятой попытки адрес заблокирован. В режиме NAT все подключения с хоста приходят с адреса `10.0.2.2`, поэтому в списке заблокированных — этот адрес.

```bash
sudo fail2ban-client status sshd                   # Banned IP list: 10.0.2.2
sudo fail2ban-client set sshd unbanip 10.0.2.2     # снятие блокировки
```

![fail2ban заблокировал адрес](images/07-fail2ban-ban.png)

---

## 3. Управление файловыми системами

### 3.1. Подключение нового диска

В VirtualBox при выключенной ВМ: «Настроить → Носители → контроллер → Добавить жёсткий диск → Создать», формат VDI, динамический, **5 ГБ**.

![Создание диска в VirtualBox](images/08-vbox-new-disk.png)

После запуска ВМ новый диск определился как `/dev/sdb`:

```bash
lsblk
sudo fdisk -l /dev/sdb
```

![Новый диск в lsblk](images/09-lsblk-new-disk.png)

### 3.2. Разметка, форматирование, монтирование

```bash
sudo fdisk /dev/sdb
#   g  — создать таблицу разделов GPT
#   n  — новый раздел (номер, начало и конец — по умолчанию, весь диск)
#   p  — показать результат
#   w  — записать изменения и выйти

sudo mkfs.ext4 /dev/sdb1
sudo mkdir -p /mnt/storage
sudo mount /dev/sdb1 /mnt/storage
df -h /mnt/storage
```

ext4 — журналируемая файловая система: изменения метаданных сначала записываются в журнал, поэтому после сбоя ФС быстро восстанавливается.

![Разметка и форматирование диска](images/10-fdisk-mkfs.png)

### 3.3. Автоматическое монтирование

Диск указывается в `/etc/fstab` по UUID: имя `sdb` может измениться при добавлении других дисков, а UUID записан в самой файловой системе.

```bash
sudo blkid /dev/sdb1
UUID=$(sudo blkid -s UUID -o value /dev/sdb1)
echo "UUID=$UUID  /mnt/storage  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab
cat /etc/fstab
```

Поля строки: устройство, точка монтирования, тип ФС, опции (`nofail` — не прерывать загрузку, если диск отсутствует), dump, порядок проверки fsck (`2` — после корневой ФС).

```bash
sudo umount /mnt/storage
sudo systemctl daemon-reload
sudo mount -a            # отсутствие ошибок подтверждает корректность строки
findmnt /mnt/storage
sudo reboot
df -h /mnt/storage       # после перезагрузки диск смонтирован автоматически
```

![Строка в fstab и проверка после перезагрузки](images/11-fstab.png)

---

## 4. Мониторинг дискового пространства

### 4.1. Команды анализа

| Команда | Что показывает                                                        | Примеры                                         |
| ------- | --------------------------------------------------------------------- | ----------------------------------------------- |
| `df`    | занятое и свободное место на смонтированных файловых системах         | `df -h`, `df -hT` (тип ФС), `df -i` (inode)     |
| `du`    | размер каталогов и файлов                                             | `du -sh /var/log`, `sudo du -h --max-depth=1 /` |
| `lsblk` | дерево блочных устройств: диски, разделы, размеры, точки монтирования | `lsblk`, `lsblk -f` (ФС и UUID)                 |
| `fdisk` | таблицы разделов и их редактирование                                  | `sudo fdisk -l`                                 |

```bash
df -h
df -hT
sudo du -sh /var/log /home
sudo du -h --max-depth=1 / 2>/dev/null | sort -h | tail
lsblk -f
sudo fdisk -l
```

`df` показывает заполненность файловых систем целиком, `du` подсчитывает размер конкретных каталогов, обходя файлы.

![df, du, lsblk, fdisk](images/12-disk-commands.png)

### 4.2. Скрипт оповещения

Скрипт `/usr/local/bin/disk_alert.sh`:

```bash
#!/bin/bash
# Предупреждает, если файловая система заполнена на THRESHOLD% и больше
THRESHOLD=80
LOG=/var/log/disk_alert.log

df -P -x tmpfs -x devtmpfs -x efivarfs | awk 'NR>1 {gsub("%","",$5); print $5, $6}' |
while read -r usage mount; do
    if [ "$usage" -ge "$THRESHOLD" ]; then
        MSG="ВНИМАНИЕ: $mount заполнен на ${usage}% (порог ${THRESHOLD}%)"
        echo "$(date '+%F %T') $MSG" >> "$LOG"   # запись в собственный журнал
        logger -t disk_alert "$MSG"              # запись в системный журнал
        wall "$MSG"                              # сообщение на терминалы пользователей
    fi
done
```

`df -P` выводит данные в стабильном формате, виртуальные ФС исключены. `awk` пропускает заголовок, убирает знак `%` и выводит процент заполнения и точку монтирования. Цикл сравнивает процент с порогом и при превышении записывает сообщение в журнал и выводит его пользователям.

Запуск каждые 15 минут через cron:

```bash
sudo chmod +x /usr/local/bin/disk_alert.sh
sudo crontab -e
```

```
*/15 * * * * /usr/local/bin/disk_alert.sh
```

![Скрипт и задание cron](images/13-disk-alert-script.png)

Проверка: диск `/mnt/storage` заполнен файлом больше чем на 80%.

```bash
sudo fallocate -l 4.2G /mnt/storage/bigfile
df -h /mnt/storage
sudo /usr/local/bin/disk_alert.sh
cat /var/log/disk_alert.log
journalctl -t disk_alert
sudo rm /mnt/storage/bigfile
```

![Срабатывание оповещения](images/14-disk-alert-test.png)

---

## Вывод

Настроен межсетевой экран UFW: входящие подключения запрещены, кроме SSH, HTTP и HTTPS; правила проверены через `ufw status` и `iptables`. Включена автоматическая установка обновлений безопасности (unattended-upgrades), настроен fail2ban для защиты SSH от подбора паролей с чтением событий из journald — блокировка проверена на практике. Подключён новый виртуальный диск 5 ГБ, создан раздел, файловая система ext4, диск смонтирован в `/mnt/storage` и добавлен в `/etc/fstab` по UUID для автоматического монтирования. Изучены команды `df`, `du`, `lsblk`, `fdisk`; написан скрипт, который при заполнении диска на 80% записывает предупреждение в журналы и выводит его пользователям, запуск выполняется через cron каждые 15 минут.
