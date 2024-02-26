## Сканирование безопасности

После создания образа рекомендуется сканировать его на наличие уязвимостей безопасности с помощью команды `docker scan`.
Docker заключил партнерское соглашение с [Snyk](http://snyk.io) для предоставления услуги сканирования уязвимостей.

Например, чтобы отсканировать образ `getting-started`, который вы создали ранее в этом руководстве, вы можете просто ввести

```bash
docker scan getting-started
```

При сканировании используется постоянно обновляемая база данных уязвимостей, 
поэтому результат, который вы увидите, будет меняться по мере обнаружения новых уязвимостей, 
но он может выглядеть примерно так:

```plaintext
✗ Low severity vulnerability found in freetype/freetype
  Description: CVE-2020-15999
  Info: https://snyk.io/vuln/SNYK-ALPINE310-FREETYPE-1019641
  Introduced through: freetype/freetype@2.10.0-r0, gd/libgd@2.2.5-r2
  From: freetype/freetype@2.10.0-r0
  From: gd/libgd@2.2.5-r2 > freetype/freetype@2.10.0-r0
  Fixed in: 2.10.0-r1

✗ Medium severity vulnerability found in libxml2/libxml2
  Description: Out-of-bounds Read
  Info: https://snyk.io/vuln/SNYK-ALPINE310-LIBXML2-674791
  Introduced through: libxml2/libxml2@2.9.9-r3, libxslt/libxslt@1.1.33-r3, nginx-module-xslt/nginx-module-xslt@1.17.9-r1
  From: libxml2/libxml2@2.9.9-r3
  From: libxslt/libxslt@1.1.33-r3 > libxml2/libxml2@2.9.9-r3
  From: nginx-module-xslt/nginx-module-xslt@1.17.9-r1 > libxml2/libxml2@2.9.9-r3
  Fixed in: 2.9.9-r4
```

В выходных данных указан тип уязвимости, URL-адрес для получения дополнительной информации 
и, что немаловажно, какая версия соответствующей библиотеки устраняет уязвимость. 

Существует несколько других вариантов, о которых вы можете прочитать в [документации по сканированию Docker](https://docs.docker.com/engine/scan/).

Помимо сканирования только что созданного образа в командной строке, 
вы также можете [настроить Docker Hub](https://docs.docker.com/docker-hub/vulnerability-scanning/) для автоматического сканирования всех вновь отправленных образов 
и затем вы сможете увидеть результаты как в Docker Hub, так и в Docker Desktop.

![Hub vulnerability scanning](hvs.png){: style=width:75% }
{: .text-center }

## Наложение образов

`Слои образов`, `Многоуровневое представление`

Знаете ли вы, что можно посмотреть, как создается образ? 
Используя команду `docker image history` команды, вы можете увидеть 
команду, которая использовалась для создания каждого слоя в образе.

1. Используйте команду `docker image history`, чтобы просмотреть слои в образе `getting-started`, 
который вы создали ранее в этом руководстве.

    ```bash
    docker image history getting-started
    ```

    Вы должны получить примерно такой результат (даты/идентификаторы могут отличаться).

    ```plaintext
    IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
    05bd8640b718   53 minutes ago   CMD ["node" "src/index.js"]                     0B        buildkit.dockerfile.v0
    <missing>      53 minutes ago   RUN /bin/sh -c yarn install --production # b…   83.3MB    buildkit.dockerfile.v0
    <missing>      53 minutes ago   COPY . . # buildkit                             4.59MB    buildkit.dockerfile.v0
    <missing>      55 minutes ago   WORKDIR /app                                    0B        buildkit.dockerfile.v0
    <missing>      10 days ago      /bin/sh -c #(nop)  CMD ["node"]                 0B        
    <missing>      10 days ago      /bin/sh -c #(nop)  ENTRYPOINT ["docker-entry…   0B        
    <missing>      10 days ago      /bin/sh -c #(nop) COPY file:4d192565a7220e13…   388B      
    <missing>      10 days ago      /bin/sh -c apk add --no-cache --virtual .bui…   7.85MB    
    <missing>      10 days ago      /bin/sh -c #(nop)  ENV YARN_VERSION=1.22.19     0B        
    <missing>      10 days ago      /bin/sh -c addgroup -g 1000 node     && addu…   152MB     
    <missing>      10 days ago      /bin/sh -c #(nop)  ENV NODE_VERSION=18.12.1     0B        
    <missing>      11 days ago      /bin/sh -c #(nop)  CMD ["/bin/sh"]              0B        
    <missing>      11 days ago      /bin/sh -c #(nop) ADD file:57d621536158358b1…   5.29MB 
    ```

    Каждая строка представляет слой образа. Здесь показано основание внизу и самый новый слой вверху. 
    Используя это, вы также можете быстро увидеть размер каждого слоя, что помогает диагностировать большие образы.

1. Вы заметите, что некоторые строки обрезаны. Если вы добавите флаг `--no-trunc`, вы получите полный вывод 

    ```bash
    docker image history --no-trunc getting-started
    ```

## Кэширование слоев

Теперь, когда вы увидели многоуровневое представление в действии, 
необходимо усвоить важный урок, который поможет сократить время сборки 
образов контейнеров. 

> После изменения слоя все последующие слои также необходимо создать заново.

Давайте еще раз посмотрим на Dockerfile, который мы использовали...

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY . .
RUN yarn install --production
CMD ["node", "src/index.js"]
```

Возвращаясь к выводу истории образа, мы видим, что каждая команда в Dockerfile становится новым слоем образа.
Возможно, вы помните, что когда мы вносили изменения в образ, `yarn`-зависимости пришлось переустанавливать. 
Есть ли способ это исправить? Нет особого смысла использовать одни и те же зависимости каждый раз при сборке, не так ли? 

Чтобы это исправить, нам нужно реструктурировать наш Dockerfile, чтобы обеспечить поддержку кэширования зависимостей. 
Для приложений на базе Node эти зависимости определяются в файле `package.json`. 
А что, если мы начнем с копирования только этого файла, установим зависимости, а затем _затем_ 
скопируем все остальное? Затем мы воссоздаем `yarn`-зависимости только в том случае, если в `package.json` были внесены изменения. Имеет смысл?

1. Обновите файл Dockerfile, чтобы сначала скопировать его в `package.json`, установить зависимости, а затем скопировать все остальное.

    ```dockerfile hl_lines="3 4 5"
    FROM node:18-alpine
    WORKDIR /app
    COPY package.json yarn.lock ./
    RUN yarn install --production
    COPY . .
    CMD ["node", "src/index.js"]
    ```

1. Создайте файл с именем `.dockerignore` в той же папке, что и файл Dockerfile, со следующим содержимым.

    ```ignore
    node_modules
    ```

    Файлы `.dockerignore` - это простой способ выборочного копирования только файлов, соответствующих образам. 
    Вы можете прочитать больше об этом [здесь](https://docs.docker.com/engine/reference/builder/#dockerignore-file).
    В этом случае папку `node_modules` следует опустить на втором этапе `COPY`, поскольку в противном случае она может перезаписать файлы, созданные командой на этапе `RUN`.
    Для получения дополнительной информации о том, почему это рекомендуется для приложений Node.js, 
    а также о дальнейших передовых практиках, ознакомьтесь с их руководством по [Докеризация веб-приложения Node.js](https://nodejs.org/en/docs/guides/nodejs-docker-webapp/).

1. Создайте новый образ, используя `docker build`.

    ```bash
    docker build -t getting-started .
    ```

    Вы должны увидеть такой вывод...

    ```plaintext
    [+] Building 16.1s (10/10) FINISHED
    => [internal] load build definition from Dockerfile                                               0.0s
    => => transferring dockerfile: 175B                                                               0.0s
    => [internal] load .dockerignore                                                                  0.0s
    => => transferring context: 2B                                                                    0.0s
    => [internal] load metadata for docker.io/library/node:18-alpine                                  0.0s
    => [internal] load build context                                                                  0.8s
    => => transferring context: 53.37MB                                                               0.8s
    => [1/5] FROM docker.io/library/node:18-alpine                                                    0.0s
    => CACHED [2/5] WORKDIR /app                                                                      0.0s
    => [3/5] COPY package.json yarn.lock ./                                                           0.2s
    => [4/5] RUN yarn install --production                                                           14.0s
    => [5/5] COPY . .                                                                                 0.5s 
    => exporting to image                                                                             0.6s 
    => => exporting layers                                                                            0.6s 
    => => writing image sha256:d6f819013566c54c50124ed94d5e66c452325327217f4f04399b45f94e37d25        0.0s 
    => => naming to docker.io/library/getting-started                                                 0.0s
    ```

    Вы увидите, что все слои были перестроены. Прекрасно, поскольку мы немного изменили Dockerfile.
  
1. Теперь внесите изменения в файл `src/static/index.html` (например, измените `<title>` на "The Awesome Todo App").

1. Теперь создайте образ Docker, снова используя `docker build -t getting-started .`. 
На этот раз ваш результат должен выглядеть немного иначе.

    ```plaintext hl_lines="10 11 12"
    [+] Building 1.2s (10/10) FINISHED
    => [internal] load build definition from Dockerfile                                               0.0s
    => => transferring dockerfile: 37B                                                                0.0s
    => [internal] load .dockerignore                                                                  0.0s
    => => transferring context: 2B                                                                    0.0s
    => [internal] load metadata for docker.io/library/node:18-alpine                                  0.0s
    => [internal] load build context                                                                  0.2s
    => => transferring context: 450.43kB                                                              0.2s
    => [1/5] FROM docker.io/library/node:18-alpine                                                    0.0s
    => CACHED [2/5] WORKDIR /app                                                                      0.0s
    => CACHED [3/5] COPY package.json yarn.lock ./                                                    0.0s
    => CACHED [4/5] RUN yarn install --production                                                     0.0s
    => [5/5] COPY . .                                                                                 0.5s
    => exporting to image                                                                             0.3s
    => => exporting layers                                                                            0.3s
    => => writing image sha256:91790c87bcb096a83c2bd4eb512bc8b134c757cda0bdee4038187f98148e2eda       0.0s
    => => naming to docker.io/library/getting-started                                                 0.0s
    ```

    Во-первых, вы должны заметить, что сборка прошла НАМНОГО быстрее! Вы увидите, что в нескольких шагах используются ранее кэшированные слои. 
    Итак, ура! Мы используем кэш сборки. Передача и извлечение этого образа и его обновлений также будет происходить намного быстрее. Ура!

## Многоэтапные сборки

Хотя мы не собираемся слишком углубляться в это в этом уроке, многоэтапные 
сборки - это невероятно мощный инструмент, который помогает нам 
использовать несколько этапов для создания образа. Они предлагают ряд 
преимуществ, в том числе: 

- Отделение зависимостей времени сборки от зависимостей времени выполнения.
- Уменьшите общий размер образа, поставляя _только_ то, что необходимо для запуска вашего приложения.

### Пример Maven/Tomcat

При создании приложений на основе Java необходима JDK для компиляции исходного кода в байт-код Java. 
Однако этот JDK не нужен в производстве. Вы также можете использовать такие инструменты, как Maven или Gradle, для создания приложения. 
Они также не нужны в нашем окончательном образе. Многоэтапные сборки помогают. 

```dockerfile
FROM maven AS build
WORKDIR /app
COPY . .
RUN mvn package

FROM tomcat
COPY --from=build /app/target/file.war /usr/local/tomcat/webapps 
```

В этом примере мы используем один этап (называемый `build`) для выполнения фактической сборки Java с помощью Maven. 
На втором этапе (начиная с `FROM tomcat`) мы копируем файлы с этапа `build`. 
Окончательный образ - это только последний создаваемый этап (который можно переопределить с помощью флага `--target`).


### React Example

When building React applications, we need a Node environment to compile the JS code (typically JSX), SASS stylesheets,
and more into static HTML, JS, and CSS. Although if we aren't performing server-side rendering, we don't even need a Node environment
for our production build. Why not ship the static resources in a static nginx container?

### Пример React

При создании приложений React нам нужна среда Node для компиляции кода JS (обычно JSX), таблиц стилей SASS и т.д. в статический HTML, JS и CSS. 
Хотя, если мы не выполняем рендеринг на стороне сервера, нам даже не понадобится среда Node для нашей производственной сборки. 
Почему бы не отправить статические ресурсы в статический контейнер nginx?

```dockerfile
FROM node:18 AS build
WORKDIR /app
COPY package* yarn.lock ./
RUN yarn install
COPY public ./public
COPY src ./src
RUN yarn run build

FROM nginx:alpine
COPY --from=build /app/build /usr/share/nginx/html
```

Здесь мы используем образ `node:18` для выполнения сборки (максимальное кэширование слоев), 
а затем копируем выходные данные в контейнер nginx. Круто, да?

## Резюме

Немного поняв, как структурированы образы, мы сможем создавать образы быстрее и вносить меньше изменений. 
Сканирование образов дает нам уверенность в том, что контейнеры, которые мы запускаем и распространяем, безопасны. 
Многоэтапные сборки также помогают нам уменьшить общий размер образа и повысить безопасность конечного контейнера 
за счет отделения зависимостей времени сборки от зависимостей времени выполнения.
