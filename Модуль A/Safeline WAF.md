Официально разворачивается в докере. Можно на отдельной ВМке
```
apt install docker docker-compose
systemctl enable --now docker
```
```
git clone https://github.com/chaitin/SafeLine.git
cd SafeLine
docker-compose up -d
```
Он займет 80/443, а backend на другом порту (8080 напрмер)
отдельно надо настроить firewall
```
firewall-cmd -add-service=http --permanent
firewall-cmd -add-service=https --permanent
firewall-cmd --reload
```
или если нестандартно:
```
firewall-cmd --add-port=80/tcp --add-port=443/tcp --permanent
```

firewall-cmd --reload
будет веб-интерфейс
добавляем сайт или приложение и порт для прослушивания
например app.greenlab.rst и порт 80
Указываем backend (upstream)
backend адрес
```
http://SRV-WEB:8080
```
Важно, что nginx на SRV-WEB должен слушать не 80/443, а другой порт, для этого
```
nano /etc/nginx/nginx.conf
```
меняем в конфиге listen 80; на другой порт
```
nginx -t
systemctl reload nginx
```
**HTTPS И СЕРТИФИКАТЫ (ОЧЕНЬ ЧАСТО В ЗАДАНИЯХ)**
### вариант А (чаще):
- HTTPS **на WAF**
- между WAF и backend — HTTP
Клиент —HTTPS→ SafeLine —HTTP→ Nginx
Сертификат выпускаем, далее
```
mkdir -p /etc/ssl/safeline
```
Кладёшь туда **3 файла**
```
/etc/ssl/safeline/
 ├─ app.greenlab.rst.crt   # сертификат
 ├─ app.greenlab.rst.key   # приватный ключ
 └─ ca.crt                 # CA FreeIPA
```
```
chmod 600 /etc/ssl/safeline/*.key
```
Проверка валидности серта
```
openssl x509 -in /etc/ssl/safeline/app.greenlab.rst.crt -noout -text
```
дальше заходим в веб админку 
```
Settings / Certificates / SSL
или
HTTPS / Certificates
```
и загружаем сертификаты, ключи
далее идем в настройки сайта, выбираем https, сертификат и порт для прослушивания 443
проверяем что сайт работаем, пробуем закинуть пэйлоад, чтоб проверить отбивает его или нет