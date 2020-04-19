# Todo api

**Авторы: Токар Илья (Икбо-05-18), Топоркова Ольга (Икбо-02-18)**
Приложение для работы с To-Do состоит из следующих микросервисов:

  - Сервис регистрации авторизации [AuthApi](https://github.com/ilyatokar/AuthApi) 
  - Сервис заданий [TodoListApi](https://github.com/ilyatokar/TodoListApi)
  - Сервис статусов заданий [ToDoSecondServer](https://github.com/SpotlessRadiance/ToDoSecondServer)

Запросы выполняются по протоколу HTTP. Для хранения данных используется реляционная база данных PostgreSQL. 
## Предустановленные программы

  - PostgreSQL;
  - сервер IIS (для развертывания на Windows);
  - сервер nginx (для развертывания на Linux);
  - пакет SDK для .NET Core 3.1


## Конфигурирование базы данных

Если сервисы запускаются в первый раз, необходимо сконфигурировать базы данных для хранения данных. БД могут храниться как на одном компьютере, так и на разных. Параметры доступа к базе данных задаются в файле **appsettings.json** строкой
`ConnectionStrings": {"DataAccessPostgreSqlProvider": "User ID=[логин];Password=[пароль];Host=[localhost];Port=[5432];Database=[название базы данных];Pooling=true;}`

Для автоматического создания пустой базы данных необходимо выполнить следующие команды:
```sh
 dotnet ef migrations add InitialCreate
 dotnet ef database update
```





## Начало работы

Для корректной работы необходимо, чтобы одновременно работали все три сервера. 
Запуск сервера осуществляется командой

Официальна документаци по развертыванию сервисов на .NET core 3.1

https://docs.microsoft.com/ru-ru/aspnet/core/host-and-deploy/linux-nginx?view=aspnetcore-3.1

### Настройка nginx

Чтобы настроить Nginx как обратный прокси-сервер для перенаправления запросов в наше приложение ASP.NET Core, измените файл /etc/nginx/sites-available/default. Откройте этот файл в текстовом редакторе и замените его содержимое на следующий код.

```
location /api/todo {
      proxy_pass         http://localhost:5000;
      proxy_http_version 1.1;
      proxy_set_header   Upgrade $http_upgrade;
      proxy_set_header   Connection keep-alive;
      proxy_set_header   Host $host;
      proxy_cache_bypass $http_upgrade;
      proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
  }

  location /api/auth {
      proxy_pass         http://localhost:4000;
      proxy_http_version 1.1;
      proxy_set_header   Upgrade $http_upgrade;
      proxy_set_header   Connection keep-alive;
      proxy_set_header   Host $host;
      proxy_cache_bypass $http_upgrade;
      proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
  }

  location /api/todostatus {
      proxy_pass         http://localhost:3000;
      proxy_http_version 1.1;
      proxy_set_header   Upgrade $http_upgrade;
      proxy_set_header   Connection keep-alive;
      proxy_set_header   Host $host;
      proxy_cache_bypass $http_upgrade;
      proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
      proxy_set_header   X-Forwarded-Proto $scheme;
  }
```

### Настройка автозпуска на сервере Ubuntu
Создайте файл определения службы. Для правильной работы сервисов нужно прописать конфиги.

```sh
sudo touch /etc/systemd/system/kestrel-helloapp.service
```
Далее представлен пример файла службы для нашего приложения.
```
[Unit]
Description=Example .NET Web API App running on Ubuntu

[Service]
WorkingDirectory=/var/www/helloapp
ExecStart=/usr/bin/dotnet /var/www/helloapp/helloapp.dll
Restart=always
# Restart service after 10 seconds if the dotnet service crashes:
RestartSec=10
KillSignal=SIGINT
SyslogIdentifier=dotnet-example
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production
Environment=DOTNET_PRINT_TELEMETRY_MESSAGE=false

[Install]
WantedBy=multi-user.target
```


```sh
dotnet run --project ./[path]/[name].csproj
```

## Сервис авторизации

Сервис осуществляет подключение пользователя. Сервис хранит информацию о зарегистрированных пользователях.

| Метод | Маршрут | Описание |
| ------ | ------ |------ |
| POST | token |В теле необходимо указать логин и пароль пользователя. Возвращает JWT-токен, который необходимо прикреплять к запросам к остальным сервисам|
| POST | - | Добавляет пользователя с указанными в теле данными|
| POST | getlogin | Получить данные о пользователе в формате JSON|

**Пример запросов:**
POST https://ltodolist.home/api/Auth/getlogin 

## Сервис заданий

Сервис осуществляет добавление, удаление и редактирование названия и описания ToDo. Для получения статуса и времени последнего его изменения сервис направляет запрос к сервису статусов заданий. К каждому запросу необходимо прикреплять заголовок Authorization с полученным токеном. Для проверки валидности токена сервис направляет запрос сервису авторизации и получает UserId пользователя. Для DELETE, GET, PUT запросов осуществляется проверка, имеет ли пользователь c данным UserId право осуществлять какие-либо действия с заданием по указанному id. 

| Метод | Маршрут | Результат |
| ------ | ------ |------ |
| GET | - |Возвращает все ToDo пользователя |
| GET | {id} |Возвращает ToDo c выбранным id (название, описание, статус)|
| POST | - |Добавляет ToDo с заданными в теле запроса параметрами(название и описание), возвращает id созданного задания |
| PUT | - |Изменяет название и/или описание задания |
| DELETE | {id} | Удаляет ToDo с выбранным id |

**Пример запросов:**
DELETE https://ltodolist.home/api/TodoItems/12 
GET https://ltodolist.home/api/TodoItems/4 


## Сервис статусов заданий
К каждому запросу необходимо прикреплять заголовок Authorization с полученным токеном (аналогично запросам к сервису заданий), так что пользователь работает только со своими заданиями. Через запросы к этому сервису можно управлять меткой выполненности(статусом) задания.

| Метод | Маршрут | Результат |
| ------ | ------ |------ |
| GET | {id} |Возвращает информацию о задании с данным id (статус и время его последнего изменения) |
| GET | history/{id} |Возвращает историю изменения статуса задания с данным id |
| GET | completed/{id} |Возвращает, было ли выполнено задание с данным id |
| GET | - |Возвращает все задания пользователя |
| POST | new/{taskId} |Добавляет статус для задания с taskId |
| PUT | {id} |Инвертирует метку выполненности задания с данным id |
| DELETE | {id} | Удаляет статус и историю изменений задания |

**Пример запросов:**
DELETE https://ltodolist.home/api/todostatus/12
POST https://ltodolist.home/api/todostatus/new/4 


