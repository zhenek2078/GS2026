```
apt install rsyslog -y
systemctl enable --now rsyslog
```
# Сервер Syslog
Включить прием логов
```
nano /etc/rsyslog.conf
```
```
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

**Дальше складываем логи по хостам, в conf добавляем**
```
$template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"
*.* ?RemoteLogs
& stop
```
и делаем
```
mkdir -p /var/log/remote
systemctl restart rsyslog
```
Проверяем
```
ss -lntup | grep 514
```
**Если нужны логи по каталогам**
```
$template PerHost,"/var/log/remote/%HOSTNAME%/%PROGRAMNAME%.log"
*.* ?PerHost
& stop
```
**Если нужна фильтрация по facility / severity (системны логи отдельно, приложения отдельно)**
```
auth,authpriv.*   /var/log/remote/auth.log
*.err             /var/log/remote/errors.log
local7.*          /var/log/remote/nginx.log
```
# Клиенты
Просто добавляем в конец конфига rsyslog.conf
**Передавать всё подряд**
```
*.* @SRV-COL:514
```
или
```
*.* @@SRV-COL:514
```
`@` - UDP  
`@@` - TCP

**Только определенные сервисы**
```
authpriv.* @SRV-COL
cron.*     @SRV-COL
```
**Отдельный файл**
```
input(type="imfile" - чтение по строкам
      File="/var/log/myapp.log" - путь к файлу
      Tag="myapp"
      Severity="info" - уровень "важности"
      Facility="local6")
```
уровни "важности"
```
emerg
alert
crit
err
warning
notice
info
debug
```
**Настраиваем передачу NGINX**
Проверяем что порт открыт на сервере
```
ss -lnt | grep 514
firewall-cmd --list-ports
```
Если файрволл задушил, то
```
firewall-cmd --add-port=514/tcp --permanent
firewall-cmd --reload
```
Открываем конфиг nginx
```
nano /etc/nginx/nginx.conf
```
Ищем блок про http
```
http {
    include       mime.types;
    default_type  application/octet-stream;

    access_log  /var/log/nginx/access.log;
    error_log   /var/log/nginx/error.log;

    include /etc/nginx/conf.d/*.conf;
}
```
МЕНЯЕМ `access_log` И `error_log`
было
```
access_log  /var/log/nginx/access.log;
error_log   /var/log/nginx/error.log;
```
стало
```
access_log syslog:server=SRV-COL:514,facility=local7,tag=nginx_access;
error_log  syslog:server=SRV-COL:514,facility=local7,tag=nginx_error;
```
- `SRV-COL` — имя или IP лог-сервера
- `facility=local7` — просто метка (можно local0–local7)
- `tag=` — чтобы на сервере было понятно, что это nginx
Проверяем конфиг nginx
```
nginx -t
```
```
systemctl reload nginx
```
далее можно сделать курс и посмотреть на серваке логи
**Запасной вариант, если не получилось напрямую передавать**
**nginx → файл → rsyslog**
На клиенте оставляем в конфиге nginx эти строки
```
access_log  /var/log/nginx/access.log;
error_log   /var/log/nginx/error.log;
```
в конфиге rsyslog на клиенте делаем
```
input(type="imfile"
      File="/var/log/nginx/access.log"
      Tag="nginx")
*.* @@SRV-COL:514
```
# firewall
```
firewall-cmd --add-port=514/udp --permanent
firewall-cmd --add-port=514/tcp --permanent
firewall-cmd --reload
```