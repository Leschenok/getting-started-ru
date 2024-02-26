
[Docker Compose](https://docs.docker.com/compose/) - это инструмент, разработанный для определения и совместного использования многоконтейнерных приложений. 
С помощью Compose мы можем создать файл YAML для определения сервисов и с помощью одной команды все раскрутить или разрушить. 

_Большим_ преимуществом использования Compose является то, что вы можете определить стек вашего приложения в файле, 
хранить его в корне репозитория вашего проекта (теперь он контролируется версиями) и легко позволить кому-то другому внести свой вклад в ваш проект. 
Кому-то нужно будет только клонировать ваш репозиторий и запустить приложение для создания сообщений. 
Фактически, вы можете увидеть довольно много проектов на GitHub/GitLab, которые сейчас делают именно это.

Итак, как нам начать?

## Установка Docker Compose

Если вы установили Docker Desktop для Windows, Mac или Linux, у вас уже есть Docker Compose!
В экземплярах Play-with-Docker уже установлен Docker Compose. Если вы 
использует другую систему, вы можете установить Docker Compose, используя [эти инструкции](https://docs.docker.com/compose/install/).

## Создание нашего файла Compose

1. Внутри папки приложения создайте файл с именем `docker-compose.yml` (рядом с файлами `Dockerfile` и `package.json`).

1. В файле компоновки мы начнем с определения списка сервисов (или контейнеров), которые мы хотим запускать как часть нашего приложения.

    ```yaml
    services:
    ```

А теперь мы начнем поочередно переносить сервисы в файл компоновки.

## Определение службы приложений

Напомним, что эту команду мы использовали для определения контейнера нашего приложения.

```bash
docker run -dp 3000:3000 \
  -w /app -v "$(pwd):/app" \
  --network todo-app \
  -e MYSQL_HOST=mysql \
  -e MYSQL_USER=root \
  -e MYSQL_PASSWORD=secret \
  -e MYSQL_DB=todos \
  node:18-alpine \
  sh -c "yarn install && yarn run dev"
```

1. Сначала давайте определим вход в службы и образ для контейнера. Мы можем выбрать любое имя для сервиса.
    Имя автоматически станет псевдонимом сети, что будет полезно при определении нашей службы MySQL.

    ```yaml hl_lines="2 3"
    services:
      app:
        image: node:18-alpine
    ```

1. Обычно вы увидите команду, близкую к определению `image`, хотя порядок расположения не требуется. 
   Итак, давайте продолжим и переместим это в наш файл.

    ```yaml hl_lines="4"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
    ```

1. Давайте перенесем часть команды `-p 3000:3000`, определив `ports` для службы. 
Здесь мы будем использовать [короткий](https://docs.docker.com/compose/compose-file/#short-syntax-2) синтаксис, 
но есть и более подробный [длинный](https://docs.docker.com/compose/compose-file/#long-syntax-2) синтаксис также доступен.

    ```yaml hl_lines="5 6"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
          - 3000:3000
    ```

1. Далее мы перенесем как рабочий каталог (`-w /app`), так и сопоставление томов (`-v "$(pwd):/app"`), используя определения `working_dir` и `volumes`.
   Тома (Volumes) также имеет [короткий](https://docs.docker.com/compose/compose-file/#short-syntax-4) 
   и [длинный](https://docs.docker.com/compose/compose-file/#long-syntax-4) синтаксис.

    Одним из преимуществ определений томов Docker Compose является то, что мы можем использовать относительные пути из текущего каталога.

    ```yaml hl_lines="7 8 9"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
          - 3000:3000
        working_dir: /app
        volumes:
          - ./:/app
    ```

1. Наконец, нам нужно перенести определения переменных среды, используя ключ `environment`.

    ```yaml hl_lines="10 11 12 13 14"
    services:
      app:
        image: node:18-alpine
        command: sh -c "yarn install && yarn run dev"
        ports:
          - 3000:3000
        working_dir: /app
        volumes:
          - ./:/app
        environment:
          MYSQL_HOST: mysql
          MYSQL_USER: root
          MYSQL_PASSWORD: secret
          MYSQL_DB: todos
    ```

  
### Определение службы MySQL

Теперь пришло время определить службу MySQL. Команда, которую мы использовали для этого контейнера, была следующей:

```bash
docker run -d \
  --network todo-app --network-alias mysql \
  -v todo-mysql-data:/var/lib/mysql \
  -e MYSQL_ROOT_PASSWORD=secret \
  -e MYSQL_DATABASE=todos \
  mysql:8.0
```

1. Сначала мы определим новую службу и назовем ее `mysql`, чтобы она автоматически получала сетевой псевдоним. 
Мы также продолжим и укажем образ, который будем использовать.

    ```yaml hl_lines="4 5"
    services:
      app:
        # The app service definition
      mysql:
        image: mysql:8.0
    ```

1. Далее мы определим сопоставление томов. Когда мы запустили контейнер с помощью `docker run`, именованный том был создан автоматически. 
Однако этого не происходит при работе с Compose. Нам нужно определить том в разделе `volumes:` верхнего уровня, а затем указать точку монтирования в конфигурации службы. 
Если просто указать только имя тома, будут использоваться параметры по умолчанию. 
Однако есть [доступно гораздо больше вариантов](https://docs.docker.com/compose/compose-file/#volumes-top-level-element).

    ```yaml hl_lines="6 7 8 9 10"
    services:
      app:
        # The app service definition
      mysql:
        image: mysql:8.0
        volumes:
          - todo-mysql-data:/var/lib/mysql
    
    volumes:
      todo-mysql-data:
    ```

1. Наконец, нам нужно указать только переменные среды.

    ```yaml hl_lines="8 9 10"
    services:
      app:
        # The app service definition
      mysql:
        image: mysql:8.0
        volumes:
          - todo-mysql-data:/var/lib/mysql
        environment: 
          MYSQL_ROOT_PASSWORD: secret
          MYSQL_DATABASE: todos
    
    volumes:
      todo-mysql-data:
    ```

На этом этапе наш полный файл `docker-compose.yml` должен выглядеть так:

```yaml
services:
  app:
    image: node:18-alpine
    command: sh -c "yarn install && yarn run dev"
    ports:
      - 3000:3000
    working_dir: /app
    volumes:
      - ./:/app
    environment:
      MYSQL_HOST: mysql
      MYSQL_USER: root
      MYSQL_PASSWORD: secret
      MYSQL_DB: todos

  mysql:
    image: mysql:8.0
    volumes:
      - todo-mysql-data:/var/lib/mysql
    environment: 
      MYSQL_ROOT_PASSWORD: secret
      MYSQL_DATABASE: todos

volumes:
  todo-mysql-data:
```

## Запуск нашего стека приложений

Теперь, когда у нас есть файл `docker-compose.yml`, мы можем его запустить!

1. Убедитесь, что другие копии приложения/базы данных не запущены (`docker ps` и `docker rm -f <ids>`).

1. Запустите стек приложения с помощью команды `docker compose up`. 
   Мы добавим флаг `-d`, чтобы все запускалось в фоновом режиме.

    ```bash
    docker compose up -d
    ```

    Когда мы запустим это, мы должны увидеть такой вывод:

    ```plaintext
    [+] Running 3/3
    ⠿ Network app_default    Created                                0.0s
    ⠿ Container app-mysql-1  Started                                0.4s
    ⠿ Container app-app-1    Started                                0.4s
    ```

    Вы заметите, что том был создан так же, как и сеть! 
    По умолчанию Docker Compose автоматически создает сеть специально для стека приложений (именно поэтому мы не определили ее в файле Compose).

1. Давайте посмотрим логи с помощью команды `docker compose logs -f`. 
Вы увидите журналы каждой службы, чередующиеся в один поток. Это невероятно 
полезно, если вы хотите следить за проблемами, связанными со временем. 
Флаг `-f` "следует" за журналом, поэтому выдает вам живую информацию по мере его создания. 

    Если вы еще этого не сделали, вы увидите вывод, который выглядит следующим образом...

    ```plaintext
    mysql_1  | 2022-11-23T04:01:20.185015Z 0 [System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.31'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306  MySQL Community Server - GPL.
    app_1    | Connected to mysql db at host mysql
    app_1    | Listening on port 3000
    ```

    Имя службы отображается в начале строки (часто цветной), чтобы помочь различать сообщения. 
    Если вы хотите просмотреть журналы определенной службы, вы можете добавить имя службы в конец команды журналов 
    (например, `docker compose logs -f app`).

    !!! info "Совет профессионала - Ожидание БД перед запуском приложения"
        Когда приложение запускается, оно фактически сидит и ждет, пока MySQL заработает и будет готов, прежде чем пытаться подключиться к нему.
        В Docker нет встроенной поддержки, позволяющей дождаться полной загрузки, запуска и готовности другого контейнера, прежде чем запускать другой контейнер. 
        Для проектов на основе Node вы можете использовать зависимость [wait-port](https://github.com/dwmkerr/wait-port). 
        Подобные проекты существуют и для других языков/фреймворков.

1. На этом этапе вы сможете открыть свое приложение и увидеть, как оно работает. И эй! У нас осталась одна команда!

## Просмотр нашего стека приложений на панели управления Docker

If we look at the Docker Dashboard, we'll see that there is a group named **app**. This is the "project name" from Docker
Compose and used to group the containers together. By default, the project name is simply the name of the directory that the
`docker-compose.yml` was located in.

Если мы посмотрим на панель управления Docker, то увидим, что есть группа с именем **app**. 
Это "имя проекта" из Docker Compose, которое используется для группировки контейнеров. 
По умолчанию имя проекта - это просто имя каталога, в котором находился `docker-compose.yml`.

![Docker Dashboard with app project](dashboard-app-project-collapsed.png)

Если вы прокрутите приложение вниз, вы увидите два контейнера, которые мы определили в файле компоновки. 
Имена также немного более информативны, поскольку они следуют шаблону `<project-name>_<service-name>_<replica-number>` (`<имя-проекта>_<имя-службы>_<номер-реплики>`). 
Таким образом, очень легко быстро увидеть, какой контейнер является нашим приложением, а какой - базой данных mysql.

![Docker Dashboard with app project expanded](dashboard-app-project-expanded.png)

## Tearing it All Down

When you're ready to tear it all down, simply run `docker compose down` or hit the trash can on the Docker Dashboard 
for the entire app. The containers will stop and the network will be removed.

## Сносим все это вниз

Когда вы будете готовы все это разобрать, просто запустите `docker compose down` или нажмите на корзину на панели управления Docker для всего приложения. 
Контейнеры остановятся, а сеть будет удалена.

!!! warning "Удаление томов"
    По умолчанию именованные тома в вашем файле компоновки НЕ удаляются при запуске `docker compose down`. 
    Если вы хотите удалить тома, вам нужно будет добавить флаг `--volumes`.

    Панель управления Docker _не_ удаляет тома при удалении стека приложения.

После демонтажа вы можете переключиться на другой проект, запустить `docker compose up` 
и быть готовым внести свой вклад в этот проект! На самом деле нет ничего проще! 

## Резюме

В этом разделе мы узнали о Docker Compose и о том, как он помогает нам 
значительно упростить определение и совместное использование 
мультисервисных приложений. Мы создали файл Compose, переведя используемые 
нами команды в соответствующий формат Compose. 

На этом этапе мы начинаем завершать урок. Однако есть несколько 
рекомендаций по созданию образов, которые мы хотим осветить, поскольку с 
используемым нами файлом Dockerfile существует большая проблема. 

Итак, давайте посмотрим!
