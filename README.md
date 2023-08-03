# infra_sprint1

## О проекте Kittygram:
Kittygram — социальная сеть для обмена фотографиями любимых питомцев.
Позволяет размещать профили котиков с указанием имени, года раждения, цветом окраса (на выбор из палитры), любыми достижениями, которые может придумать владелец.
Профиль котика может быть с фото или без.  
Редактировать профиль питомца может только добавивший его хозяин.

## Как запустить проект:
1. Сделайте fork депозитория infra_sprint1 в свой github
2. Настройте пару SHH ключей для клонирования проекта на свой сервер с github:
   
   ```
   ssh-keygen
   ```
   Сохраните файлы с ключами в директорию по умолчанию: для этого нажмите Enter на Windows или Return на macOS.
   Можно задать пароль или пропустить:
   ```
   Enter passphrase (empty for no passphrase):
   ```
   Теперь необходимо сохранить открытый ключ в вашем аккаунте на GitHub.
   Выведите ключ в терминал командой:
   ```
   cat .ssh/id_rsa.pub
   ```
   Скопируйте ключ от символов ssh-rsa, включительно, и до конца.
   Зайдите в свой аккаунт на GitHub, перейдите в раздел настроек.
   Выберите пункт SSH and GPG keys; для создания нового ключа нажмите на кнопку New SSH key в правом верхнему углу.
   Скопируйте туда полученный в терминале ключ и задайте ему подходящее имя.
3. Клонирование проекта с GitHub на сервер:
   ```
   # Вместо <ваш_аккаунт> подставьте свой логин, который используете на GitHub.
   # Вместо <название_репозитория> подставьте название репозитория,
   # который хотите клонировать.
   git clone git@github.com:ваш_аккаунт/название_репозитория.git
   # Например:
   # git clone git@github.com:yandex-practicum/infra_sprint1.git
   ```
4. Настройка бэкенд-приложения:
   Установите зависимости из файла requirements.txt:
   ```
   # Перейдите в директорию бэкенд-приложения проекта.
   # Вместо <название_проекта> укажите название проекта, с которым работаете.
   cd название_проекта/backend/
   # Создайте виртуальное окружение.
   python3 -m venv venv
   # Активируйте виртуальное окружение.
   source venv/bin/activate
   # Установите зависимости.
   pip install -r requirements.txt
   ```
   Выполните миграции и создайте суперюзера из директории с файлом manage.py:
   ```
   # Примените миграции.
   python3 manage.py migrate
   # Создайте суперпользователя.
   python3 manage.py createsuperuser
   ```
   Замените в списоке ALLOWED_HOSTS внешний IP сервера, и домен на тот который вы получили для сайта размещения проекта:
   ```
   # Вместо xxx.xxx.xxx.xxx укажите IP сервера, а вместо <ваш_домен> – доменное имя.
   ALLOWED_HOSTS = ['xxx.xxx.xxx.xxx', '127.0.0.1', 'localhost', 'ваш_домен']
   ```
   Соберите статику бэкенд-приложения:
   ```
   python3 manage.py collectstatic
   ```
   Скопируйте директорию static_backend/ в директорию /var/www/название_проекта/:
   ```
   sudo cp -r путь_к_директории_с_бэкендом/static_backend /var/www/название_проекта
   ```
5. Установка и настройка WSGI-сервера Gunicorn:
   Не выключая виртуальное окружение проекта, установите пакет gunicorn:
   ```
   pip install gunicorn==20.1.0
   ```
   Перейдите в директорию с файлом manage.py, и запустите Gunicorn на порте 8080:
   ```
   gunicorn --bind 0.0.0.0:8080 backend_название проекта.wsgi
   ```
   Создайте файл конфигурации юнита systemd для Gunicorn в директории /etc/systemd/system/.
   Назовите его по шаблону gunicorn_название_проекта.service:
   ```
   sudo nano /etc/systemd/system/gunicorn_название_проекта.service
   ```
   Скопируйте готовые настройки из файла infra_sprint1/infra/gunicorn_kittygram.serviceв файл конфигурации Gunicorn
   Замените имя пользователя (User) и рабочую дирректорию (WorkingDirectory) на свои данные и сохраните изменения.

   Запустите Gunicorn и запустите Gunicorn Daemon:
   ```
   sudo systemctl start gunicorn_название_проекта
   sudo systemctl enable gunicorn_название_проекта
   ```
   
7. Настройка фронтенд-приложения:
   Находясь в директории с фронтенд-приложением, установите зависимости для него:
   ```
   npm i
   ```
   Из директории с фронтенд-приложением выполните команду:
   ```
   npm run build
   ```
   Скопируйте статику фронтенд-приложения в директорию по умолчанию:
   ```
   sudo cp -r путь_к_директории_с_фронтенд-приложением/build/. /var/www/имя_проекта/
   ```
8. Установка и настройка веб- и прокси-сервера Nginx:
   Установите и запустите nginx
   ```
   sudo apt install nginx -y
   sudo systemctl start nginx
   ```
   Обновите настройки Nginx. Для этого откройте файл конфигурации веб-сервера
   ```
   sudo nano /etc/nginx/sites-enabled/default
   ```
   Если на сервере уже есть проекты, добавьте настройки для сервера kittigram из файла infra_sprint1/infra/default.
   _До получения ssl сертификата нужно указать "прослушивание" порта 80. И не копировать строки, добавленные cerbot._
   _Не забудьте поменять имя сервера на ваш домен!_

   ```
   # Kittigram server configuration
    server {
      listen 80;
      server_tokens off;
      server_name ash-kittygram.sytes.net;

        location /media/ {
	        client_max_body_size 20M;		
	        alias /var/www/kittygram/media/;
	      }

        location /api/ {

            proxy_pass http://127.0.0.1:8080;
            client_max_body_size 20M;
        }

        location /admin/ {
            proxy_pass http://127.0.0.1:8080;
            client_max_body_size 20M;
        }

        location / {
        root   /var/www/kittygram;
        index  index.html index.htm;
        try_files $uri /index.html;
        }
   ```
   Сохраните файл, проверьте на опечатки и перезапустите nginx
   ```
   sudo nginx -t
   sudo systemctl reload nginx
   ```
9. Настройка файрвола ufw
   ```
   sudo ufw allow 'Nginx Full'  # Открывает порты 80 и 443 для запросов http и https
   sudo ufw allow OpenSSH  # Открывает порт 22 - нужен, чтобы вы могли подключаться к серверу по SSH.
   sudo ufw enable  # Включите файрвол
   ```
10. Получение и настройка SSL-сертификата
    Находясь на сервере, установите certbot, если он ещё не установлен:
    ```
    # Установка пакетного менеджера snap.
    sudo apt install snapd
    # Установка и обновление зависимостей для пакетного менеджера snap.
    sudo snap install core; sudo snap refresh core
    # Установка пакета certbot.
    sudo snap install --classic certbot
    # Обеспечение доступа к пакету для пользователя с правами администратора.
    sudo ln -s /snap/bin/certbot /usr/bin/certbot
    ```
    Запустите certbot и получите SSL-сертификат. Для этого выполните команду:
    ```
    sudo certbot --nginx
    ```
    Далее система попросит вас указать электронную почту и ответить на несколько вопро- сов. Сделайте это.
    Следующим шагом укажите доменное имя, которое вы получили для этого проекта и для которого хотели бы активировать HTTPS:

    Проверьте конфигурацию Nginx, и если всё в порядке, перезагрузите её.
    ```
    sudo nginx -t
    sudo systemctl reload nginx
    ```
### Поздравляю теперь ваш проект доступен в интернет 
### и данные защищены шифрованием по протоколу https
   

## Примеры запросов API:
`Операции с котиками`
### Get api/cats/
Вернет список из 10 котиков на странице
```
{
    "count": 55,
    "next": "http://api.example.org/cats/?offset=50&limit=10",
    "previous": "http://api.example.org/cats/?offset=30&limit=10",
    "results": [{}]
}
```
### Post api/cats/
Опубликует профиль котика. Необязательные поля: achievements, image.
```
{
    "name": "string: max_length=16",
    "color": "string: webcolors.hex_to_name",
    "birth_year": "integer",
    "owner": 0,
    "achievements": 0,
    "image": "string(base64)"
}
```
`Операции с кошачьими достижениями`
### Get api/achievements/
Предоставит список всех достижений заданных пользователями.

### Post api/achievements/
```
{
    "id": 0,
    "name": "string: max_length=64"
}
```
`Операции с пользователями`
### Post api/users/
Регистрация пользователя на сайте
```
{
    "username": "User.USERNAME_FIELD",
    "password": "string",
    "re-password": "string"
}
```
Ответ:
```
{
    "username": "User.USERNAME_FIELD",
    "pk": "User._meta.pk.name "
}
```
### Post api/token/login/
Залогинить пользователя на сайте
```
{
    "username": "User.USERNAME_FIELD",
    "password": "string"
}
```
Ответ:
```
{
    "auth_token": "string"
}
```
### Post api/token/logout/
Разлогинить пользователя на сайте
```
{
    "username": "User.USERNAME_FIELD",
    "password": "string"
}
```
Ответ - статус HTTP_204_NO_CONTENT


## Технологии:

Backend
* Django
* djangorestframework
* joiser
* SQLite

Frontend
* JavaScript
* React

WSGI сервер
* Gunicorn

WEB сервер
* NGINX

## Об авторе:
Анастасия Андреева - бэкенд разработчик на пайтон.
`https://github.com/Comaash`
