проверка
```
rsync --version
```
# Шаг 1. Пользователь backuser
**На SRV-BCP**
```
useradd backuser
passwd backuser
mkdir -p /home/backuser
chown backuser:backuser /home/backuser
```
На клиентах
```
useradd backuser
passwd backuser
```
копирование должно выполняться УЗ backuser  
→ значит cron и rsync запускаются от backuser

# ШАГ 2. SSH БЕЗ ПАРОЛЯ
На клиентах
```
su - backuser
ssh-keygen
```
Проверить ключи
```
ls ~/.ssh
```
Передать публичный ключ на сервер
```
ssh-copy-id backuser@SRV-BCP
```
что делает эта команда:
1. логинится по паролю (ПОСЛЕДНИЙ РАЗ)
2. создаёт на сервере: /home/backuser/.ssh/authorized_keys
3. копирует туда `id_rsa.pub`
Либо переносим ключ руками
```
cat ~/.ssh/id_rsa.pub
```
Дальше на сервере
```
su - backuser
mkdir -p ~/.ssh
nano ~/.ssh/authorized_keys
```
Вставляем ключ по одной строке и даем права
```
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```
Проверка с клиента
```
ssh backuser@SRV-BCP
```
# ШАГ 3. Скрипт
```
nano /home/backuser/backup_home.sh
```
```
#!/bin/bash

# 1. имя текущего хоста
HOST=$(hostname)

# 2. текущая дата и время
DATE=$(date +"%Y-%m-%d_%H-%M-%S")

# 3. каталог назначения на сервере бэкапов
DEST="/home/backuser/${HOST}/backup_${DATE}"

# 4. создаём каталог на сервере
mkdir -p "$DEST"

# 5. копируем /home на сервер SRV-BCP
rsync -avz /home/ backuser@SRV-BCP:"$DEST"
```
```
chmod +x /home/backuser/backup_home.sh
```
Запускаем руками, проверяем что все создается и передается
Настраиваем cron
```
crontab -e
```
формат cron
```
минута  час  день_месяца  месяц  день_недели  команда
```
**cron на конкретное время**
 пример: **каждый день в 02:30**
```
30 2 * * * /home/backuser/backup_home.sh
```
 пример: **каждый день в 23:00**
```
0 23 * * * /home/backuser/backup_home.sh
```
 пример: **в 01:00 и в 13:00**
```
0 1,13 * * * /home/backuser/backup_home.sh
```
 пример: **каждые 30 минут**
```
*/30 * * * * /home/backuser/backup_home.sh
```
 пример: **каждые 2 часа**
```
0 */2 * * * /home/backuser/backup_home.sh
```