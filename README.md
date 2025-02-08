# petproject1
Пет проект, мониторинг количества подключений к базе данных.
Вариант ТЗ:
  1) Два docker-контейнера, на первом база данных PostgreSQL, на втором Grafana для мониторинга за этой базой (в нашем случае мониторинг за количеством активных подключений);
  2) В качестве веб-сервер - Nginx, проксирует запросы к Grafana, чтобы можно было наблюдать со стороны хоста;
  3) После перезагрузки контейнеров все данные должны сохраняться;

## Содержание
- [Технологии](#технологии)
- [Требования](#требования)
- [Реализация](#реализация)
- [Зачем](#зачем)
- [Создатель](#создатель)

## Технологии
- [Docker](https://www.docker.com/)
- [Nginx](https://www.nginx.org/)
- [Grafana](https://www.grafana.com/)
- [PostgreSQL](https://www.postgresql.org/)

## Требования
Для реализации проекта необходимы следующие магические штуки.

Установите docker с помощью команды (выглядит страшно, но есть вариант установки deb-пакета через dkpg):
```sh
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
Если все сделано правильно, то после ввода этой команды должен запуститься тестовый контейнер:

```sh
sudo docker run hello-world
```

C Nginx проще, он лежит в официальном репозитории, устанавливаем при помощи команды: 
```sh
sudo apt install nginx
```

## Реализация

### Создание docker-compose.yml

Создаем файл docker-compose.yml в любой удобной директории. 
Это файл Docker Compose, который будет содержать инструкции, необходимые для запуска и настройки сервисов.

В нашем случае используется мультконтейнерное приложение, поэтому дабы облегчить себе жизнь, используем именно Docker Compose.

```yml
#Пустые строки при запуске игнорируются, я их использовал для лучшей читаемости. Табуляция обязательна!!!
version: "3.8" #Версия Докера

services: #непосредственно наши контейнеры
  postgres:
    image: postgres:13 #версия
    container_name: postgres
    environment: набор переменных окружения
      POSTGRES_USER: admin #логин
      POSTGRES_PASSWORD: admin #пароль
      POSTGRES_DB: db #название базы
    volumes:
      - postgres_data:/var/lib/postgresql/data #директория, где будут сохраняться данные после выполнения контейнера
    ports:
      - "5432:5432" #дефолтный порт postgresql
    networks:
      - grafana_network #название сети, которая будет использоваться для общения между контейнерами

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    depends_on: #определение зависимости между контейнерами
      - postgres
    volumes:
      - grafana_data:/var/lib/grafana
    ports:
      - "3000:3000"
    networks:
      - grafana_network

volumes:
  postgres_data:
  grafana_data:

networks:
  grafana_network:
```
Стартанем всю эту красоту командой из директории, где лежит файлик
```sh
sudo docker compose up
```

### Настройка Nginx
Теперь создадим конфигурационный файл для проксирования запросов к Grafana.

Перейдем в директорию и создадим файлик grafana
```sh
cd /var/etc/nginx/sites-available
sudo touch grafana
```
В файлик пихаем следющие буковы:
```
server {
    listen 80;
    server_name localhost; 

    location / { 
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host; 
        proxy_set_header X-Real-IP $remote_addr; 
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

```
Делаем линк этого файлика в директорию sites-enabled
```sh
sudo ln -s /etc/nginx/sites-available/grafana /etc/nginx/sites-enabled/
```
Проверим работоспобность Nginx и перезапустим его
```sh
sudo nginx -t
sudo systemctl restart nginx
```

### Настройка дашборда Grafana
После предыдудщих действий введем в браузер:
```url
localhost
```
Попросит login/pass - введём admin/admin. Если откроется наша Grafana, то мы молодцы и правильно сконфигурировали Nginx

Затем нужно добавить так называемые _datasource_, то есть источники наших данных, над которыми мы хотим иметь контроль. 
Добавление _datasource_:
  1) Слева на панели жмем Dashboard;
  2) Вверху справа жмем New -> New dashboard;
  3) Add visualisation;
  4) Configure a new data source;
  5) Ищем в поиске PostgreSQL ;
  6) Заполни поля:
     - Name: PostgreSQL
     - Host: postgres:5432
     - Database: db
     - User: admin
     - Password: admin
     - TLS/SSL Mode - disable;
  7) Save & test;
  8) Выбираем новый _datasource_ - PostgreSQL

Откроется редактор дашборда, снизу будет вкладка Queries, меняем положение переключателя с Builder на Code.
Откроется окно ввода, куда введем sql-запрос количества активных подключений:
```sql
SELECT COUNT(*) AS active_connections
FROM pg_stat_activity
WHERE state IS NOT NULL;
```
![изображение](https://github.com/user-attachments/assets/d02833f6-53ed-4242-857a-2afab805c45a)

Меняем формат на _table_.
Справа меняем вид отображения данных на _stat_.

После проделанных операций на экране гордо высветится цифра **1**.
Grafana же - единственный, кто на данный момент подключен к этой базе.

Для теста счетчика активных соединений можем подключиться к базе из под терминала, введем команду:
```sh
sudo docker exec -it postgres bash
```
Затем:
```sh
psql -U admin db
```
Если вдруг psql не работает, устанавливаем:
```
sudo apt install postgresql postgresql-contrib
```
В дашборде Grafana должно увеличиться число активных соединений.

![изображение](https://github.com/user-attachments/assets/74005841-be24-4c87-a097-53a5646cf430)

### Остановка работы и перезапуск
Для остановки работы нашего мультиконтейнерного "приложения" введем в терминал:
```sh
docker-compose down
```
Для повторного запуска введем:
```sh
docker-compose up -d
```

## Зачем 
Как минимум для общего развития, как максимум для будущего работодателя, который посмотрит на это творения и скажет:
```
Воу, он что-то умеет
```

## Создатель
Danya Kislov aka @pylnoesolnce

![изображение](https://github.com/user-attachments/assets/1c1d1a2d-4654-4fd1-ab50-c2674a4decec)

