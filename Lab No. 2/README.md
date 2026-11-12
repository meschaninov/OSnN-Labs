# Лабораторная работа № 2. Управление системой и пользователями

Студент: Мещанинов Артём Александрович
Группа: ИС-25

## Цель работы

Научиться создавать пользователей и группы, настраивать права доступа, следить за активностью пользователей и автоматизировать создание пользователей скриптом.

Все команды выполнялись с Mac по SSH от имени root (`sudo -i`).

---

## Задание 1. Создание пользователей и групп

Созданы пользователи `admin`, `user1`, `user2` (`-m` — создать домашнюю папку, `-s` — оболочка) и заданы пароли:

```bash
useradd -m -s /bin/bash admin
useradd -m -s /bin/bash user1
useradd -m -s /bin/bash user2
passwd admin
passwd user1
passwd user2
```

![Создание пользователей](images/01-create-users.png)

Созданы группы `developers` и `testers`. В developers добавлены admin и user1, в testers — user2:

```bash
groupadd developers
groupadd testers
usermod -aG developers admin
usermod -aG developers user1
usermod -aG testers user2
id admin; id user1; id user2
getent group developers testers
```

![Создание групп](images/02-groups.png)

---

## Задание 2. Настройка прав доступа

```bash
mkdir -p /home/admin/data /home/admin/logs

chown admin:admin /home/admin/data
chmod 755 /home/admin/data

chown admin:developers /home/admin/logs
chmod 770 /home/admin/logs

chmod 711 /home/admin
ls -ld /home/admin /home/admin/data /home/admin/logs
```

- `data` — `755`: изменять файлы может только admin, остальные только читают;
- `logs` — `770`: читать и писать могут admin и группа developers, остальные — нет;
- `/home/admin` — `711`: в Debian домашняя папка закрыта для всех, без этого user1 не смог бы попасть в `logs`.

![Права доступа](images/03-permissions.png)

Проверка:

```bash
sudo -u user1 touch /home/admin/logs/test.log   # получилось
sudo -u user2 touch /home/admin/logs/t2.log     # Отказано в доступе
sudo -u user1 touch /home/admin/data/f.txt      # Отказано в доступе
sudo -u admin touch /home/admin/data/f.txt      # получилось
```

![Проверка прав](images/04-permissions-check.png)

---

## Задание 3. Команды для работы с пользователями

| Команда    | Что делает                                               |
| ---------- | -------------------------------------------------------- |
| `useradd`  | создаёт пользователя                                     |
| `usermod`  | изменяет пользователя (добавить в группу, заблокировать) |
| `groupadd` | создаёт группу                                           |
| `passwd`   | меняет пароль                                            |
| `chown`    | меняет владельца файла или папки                         |
| `chmod`    | меняет права доступа (r=4, w=2, x=1)                     |

Смена пароля user1:

```bash
passwd user1
```

![Смена пароля user1](images/05-passwd-user1.png)

Блокировка user2:

```bash
usermod -L user2
passwd -S user2
grep user2 /etc/shadow
```

`passwd -S` показывает букву `L` — учётная запись заблокирована. В файле `/etc/shadow` перед паролем появился знак `!`.

![Блокировка user2](images/06-lock-user2.png)

---

## Задание 4. Мониторинг активности пользователей

Во второй вкладке терминала выполнен вход от имени user1:

![Вход user1](images/07a-user1-login.png)

```bash
who
w
last -n 10
```

- `who` — показывает, кто сейчас в системе;
- `w` — то же, плюс время работы системы, нагрузку и что делает каждый пользователь;
- `last` — история входов в систему.

На скриншоте видно, что в системе сейчас user1 и user (по SSH с адреса `10.0.2.2` — это Mac) и user в консоли `tty1`.

![who, w, last](images/07-monitoring.png)

---

## Задание 5. Скрипт для создания пользователей

Файл `users.txt` (имя и группа):

```
ivanov developers
petrov testers
sidorova developers
```

Скрипт `create_users.sh`:

```bash
#!/bin/bash
# Создание пользователей из файла users.txt
# Формат строки: имя группа

while read -r name group; do
    if ! getent group "$group" > /dev/null; then
        groupadd "$group"
        echo "Создана группа $group"
    fi

    if id "$name" > /dev/null 2>&1; then
        echo "Пользователь $name уже существует"
    else
        useradd -m -s /bin/bash -G "$group" "$name"
        echo "Создан пользователь $name"
    fi
done < users.txt
```

Скрипт читает файл по строкам. Если группы нет — создаёт её. Если пользователя нет — создаёт и добавляет в группу.

![Скрипт](images/08-script.png)

Запуск и проверка:

```bash
chmod +x create_users.sh
./create_users.sh
id ivanov; id petrov; id sidorova
```

При повторном запуске скрипт пишет, что пользователи уже существуют.

![Работа скрипта](images/09-script-run.png)

---

## Вывод

Созданы пользователи и группы, настроены права доступа к папкам, изменён пароль user1 и заблокирован user2. Изучены команды `who`, `w`, `last`. Написан скрипт, который создаёт пользователей и группы из файла.
