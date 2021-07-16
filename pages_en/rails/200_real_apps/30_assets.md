---
title: Генерируем и раздаём ассеты
permalink: rails/200_real_apps/30_assets.html
layout: wip
---

[comment]: <> (Вводная: общая для всех фреймворков)

В этой главе мы рассмотрим особенности организации, сборки и раздачи ассетов (CSS, JS, изображений и других статических файлов).

Для повышения производительности приложения ответственность за работу со статическими файлами делегируется на отдельный веб-сервер.
В качестве этого компонента будет использоваться NGINX, который будет раздавать ассеты, а также отвечать за роутинг и выступать в роли прокси-сервера.

{% plantuml %}
package "Deployment" {
    node "Pod" {
        [frontend]
        [backend]
    }
}

cloud "Ingress"

Ingress - [frontend]
[frontend] - [backend]
{% endplantuml %}

С тем же успехом это мог быть Apache или любое другое популярное решение.

[comment]: <> (Вводная: специфика rails)

При разработке обслуживанием всех файлов приложения занимается встроенный RoR-сервер, а компиляция ассетов выполняется на лету во время обработки запроса.
Несмотря на то, что ценные ресурсы Ruby-воркера расходуются неоптимальным образом, такой подход упрощает разработку и привносит ряд преимуществ при итеративной сборке ассетов.

> В случае с webpack при разработке принято поднимать [webpack-dev-server](https://github.com/webpack/webpack-dev-server), который следит за изменениями в ассетах и при необходимости пересобирает их в фоне.

По умолчанию в production окружении (`RAILS_ENV=production`) сервер перестаёт раздавать ассеты — ожидается, что сопровождение ассетов будет выполняться другим компонентом.

## Ассеты должны быть воспроизводимыми и не собираться под окружение

К сожалению, в традиции фронтенд-разработки сложилась практика, когда на стадии сборки ассетов пробрасываются специфичные для окружения параметры, но так делать не следует.

Ассеты не должны содержать логинов, паролей, доменов и подобных данных, которые могут подставляться при компиляции в зависимости от конечного окружения.
Процесс компиляции не должен иметь внешних зависимостей, которые нарушают воспроизводимость получаемых ассетов — при генерации ассетов не нужно использовать баз данных, сгенерированный пользователем контент и т.д.

{% offtopic title="Что делать с legacy-проектами?" %}
Мы понимаем, что существует огромное количество приложений, в которых конфигурация задаётся на стадии сборки.
Конечно, лучше начать переписывать legacy раньше, чем позже. Но мы понимаем, что это дорогая и сложная задача.

В качестве временной меры _иногда_ можно подставить уникальную строку вместо завязанных на окружение значений.
К примеру, если в приложение передаётся домен CDN-сервера `cdn_server` со значениями `mycdn0.mydomain.io` / `mycdn0-staging.mydomain.io` — можно вместо этих значений при сборке использовать уникальный GUID (к примеру, `cdfe0513-ba1f-4f92-8503-48a497d98059`).
А при запуске сервера в Kubernetes первым делом выполнять подстановку значений для окружения, подменяя определённые GUID в ассетах с помощью утилиты [sed](https://ru.wikipedia.org/wiki/Sed).

Использование этого или другого «костыля» является лишь временной мерой с сомнительным результатом и не избавляет от необходимости модернизировать приложение.
{% endofftopic %}

Если эти данные не хранятся в ассетах, то в таком случае они должны получаться извне.

В этой статье будет показан пример реализации, который отвязан от самого приложения и реализуется за счёт инфраструктуры.
Все необходимые параметры будут сохраняться в ConfigMap и доступны по зарезервированному адресу, используя который можно выстраивать логику в JS-файлах.

{% plantuml %}
package "Deployment" {
    node "Pod" {
        [frontend]
        [backend]
    }
}

package "Configmap" {
  file env.json
}

cloud "Ingress"

Ingress - [frontend]
[frontend] - [backend]
[frontend] -- [env.json]
{% endplantuml %}

{% offtopic title="Как выполнять эту функцию приложением?" %}

При запуске приложения в Kubernetes обычно контекстно-зависимые параметры задаются переменными окружения, опираясь на которые можно генерировать подходящий набор параметров.
Таким образом, имея доступ к этим переменным окружения в приложении, можно реализовать специальный API-метод или даже выстроить логику в view-файле.

Рассмотрим абстрактный пример view-файла:

```html
<!DOCTYPE html>
<html>
  <head>
    <title>Application</title>
    <link rel="icon" type="image/png" href="<%= asset_path 'favicon.png' %>">

    <%= stylesheet_link_tag 'application', media: 'all' %>
  </head>
<body>
  <div id="app-root-element">

    <div id="navigator-container"></div>

    <div class="view-container">
      <div ui-view class="view-frame"></div>
    </div>

  </div>
  <script type="text/javascript">
    //<![CDATA[
    sessionStorage.setItem("current_user", '<%= @current_user_presentation.to_json.html_safe %>')
    <% if ENV['SENTRY_PUBLIC_DSN'] %>
      window.sentryPublicDsn = '<%= ENV['SENTRY_PUBLIC_DSN'] %>';
      <% if ENV['SENTRY_CURRENT_ENV'] %>
        window.sentryEnvironment = '<%= ENV['SENTRY_CURRENT_ENV'] %>';
      <% end %>
    <% end %>
    //]]>
  </script>

<%= javascript_include_tag 'application' %>
</body>
</html>
```

В данном примере в зависимости от переменных окружения `SENTRY_PUBLIC_DSN` и `SENTRY_CURRENT_ENV` в подключаемом JS-файле будут доступны определённые значения.
{% endofftopic %}

## Код приложения

С исходным кодом приложения можно ознакомиться в репозитории.

За подготовку ассетов в нашем приложении будет отвечать webpack.
Тем не менее, предложенный подход не будет существенно отличаться и для других технологий, а также в случае, если используемые ассеты не требуют компиляции.

Работоспособность предложенного подхода будет проверяться на корневой странице приложения.
Страница использует CSS-стили и JS-скрипт.
При инициализации JS-скрипт загружает параметры окружения по определённому адресу, использует их, а затем запрашивает список лейблов по API и выводит его на странице.

Для понимания того, как это всё устроено достаточно посмотреть на view-файл и подключаемый JS:

```html
<!DOCTYPE html>
<html lang="en">
    <head>
        <meta charset="UTF-8">
        <title>Labels</title>

        <%= javascript_pack_tag 'labels' %>
        <%= stylesheet_pack_tag 'styles/labels' %>
    </head>
    <body>
        <div class="content">
          <img src="<%= asset_pack_path 'media/images/werf-logo.svg' %>"  alt="werf logo"/>
          <h1>Labels in <span id="environment">NONE</span> environment!</h1>
          <p><strong>List of labels from backend: </strong><span id="content"></span></p>
          <p><strong>Link from config: </strong> <a href="#" id="link">here</a></p>
        </div>
    </body>
</html>
```

```js
var request = new XMLHttpRequest();
request.open('GET', '/config/env.json', false);  // `false` makes the request synchronous
request.send(null);

if (request.status === 200) {
    document.addEventListener('DOMContentLoaded', function(event) {
        const variables = JSON.parse(request.responseText);
        document.getElementById("environment").innerHTML = variables.environment;
        document.getElementById("link").href = variables.link;

        // Business logic here
        console.log('It works');
        var request_content = new XMLHttpRequest();
        request_content.open('GET', '/api/labels', false);  // `false` makes the request synchronous
        request_content.send(null);
        if (request_content.status === 200) {
            document.getElementById("content").innerHTML = request_content.responseText;
        } else {
            document.getElementById("content").innerHTML = "sorry, error while loading";
        }
    })
}
```

Стоит отметить, что первым шагом в JS-файле идёт запрос по адресу `/config/env.json`.
Это зарезервированный путь, по которому доступны переменные окружения.
Эти переменные будут храниться в контейнере вместе с NGINX и ассетами и монтироваться из ConfigMap при запуске (подробнее далее).

## Сборка ассетов

Вместо одного образа `basicapp` теперь будет собираться два: `backend` и `frontend`.
Для `backend` и для компиляции ассетов для `frontend` требуются общие компоненты, поэтому используется один Dockerfile и multi-stage в нём.

```yaml
project: werf-guided-rails
configVersion: 1

---
image: backend
dockerfile: Dockerfile
target: backend
---
image: frontend
dockerfile: Dockerfile
target: frontend
```

```Dockerfile
FROM ruby:2.7.1 as rails

WORKDIR /app

# Install system dependencies
RUN apt-get -o Acquire::Check-Valid-Until=false -o Acquire::Check-Date=false update -qq && apt-get install -y build-essential libpq-dev libxml2-dev libxslt1-dev curl default-mysql-client

# Install app dependencies
COPY Gemfile /app/Gemfile
COPY Gemfile.lock /app/Gemfile.lock
RUN bundle install

FROM rails as assets_builder

# Install NodeJS 14 and yarn
RUN curl -sL https://deb.nodesource.com/setup_14.x | bash -
RUN apt-get install -y nodejs
RUN npm install yarn --global

# Install assets dependencies
COPY yarn.lock package.json /app/
RUN yarn install

# Add all the files from our repo into our image, including app source code
COPY . .

# Prepare assets
RUN SECRET_KEY_BASE=NONE RAILS_ENV=production rails assets:precompile

FROM nginx:stable-alpine as frontend

# Copy all assets
COPY --from=assets_builder /app/public /www

# Add nginx configuration
COPY .werf/nginx.conf /etc/nginx/nginx.conf

FROM rails as backend

# Add all the files from our repo into our image, including app source code
COPY . .

# Add webpack manifest (precompiled assets not needed)
COPY --from=assets_builder /app/public/packs/manifest.json /app/public/packs/manifest.json
```

Описанный Dockerfile можно представить следующим образом:

{% plantuml %}
file nginx.conf
agent rails
agent assets_builder
agent backend
agent frontend

rails -- backend
rails -- assets_builder
assets_builder -> backend: manifest.json
assets_builder ---> frontend: assets
nginx.conf -> frontend
{% endplantuml %}

Исходный код `nginx.conf` можно посмотреть в репозитории.

## Изменения в инфраструктуре и роутинге

Первым делом добавим образ с ассетами в наш Deployment и направим весь трафик на него:

{% raw %}
```yaml
# examples/rails/100_assets/.helm/templates/deployment.yaml

      - name: frontend
        command: [ "/usr/sbin/nginx" ]
        image: {{ .Values.werf.image.frontend }}
        ports:
          - containerPort: 80
```
{% endraw %}

```yaml
# examples/rails/100_assets/.helm/templates/service.yaml

apiVersion: v1
kind: Service
metadata:
  name: basicapp
spec:
  selector:
    app: basicapp
  ports:
    - name: http
      port: 80
```

```yaml
# examples/rails/100_assets/.helm/templates/ingress.yaml

apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
  name: basicapp
spec:
  rules:
    - host: example.com
      http:
        paths:
          - path: /
            backend:
              serviceName: basicapp
              servicePort: 80
```

Следующим шагом создадим ConfigMap, данные которого будут раздаваться NGINX по адресу `/config/env.json`.

{% raw %}
```yaml
# examples/rails/100_assets/.helm/templates/configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: basicapp-configmap
data:
  env.json: {{ .Values.assetsValues | mustToJson | quote }}
```
{% endraw %}

При генерации json используется значение `assetsValues`, которое может быть задано с помощью values-файлов (в том числе секретных) и с помощью опций CLI.
В нашем примере мы будем использовать values-файл `.helm/values.yaml` с расчетом на то, что на данном этапе деплоится только production-окружение, данные статичны и не требуются secrets.

```yaml
# examples/rails/100_assets/.helm/values.yaml

assetsValues:
  environment: "Production"
  link: "https://werf.io"
```

Для того, чтобы NGINX начал обслуживать эти данные, необходимо примонтировать их по определённому пути в контейнер `frontend`.
В нашем случае это путь `/www/config/env.json`.

{% raw %}
```yaml
# examples/rails/100_assets/.helm/templates/deployment.yaml

      - name: frontend
        command: [ "/usr/sbin/nginx" ]
        image: {{ .Values.werf.image.frontend }}
        ports:
          - containerPort: 80
        volumeMounts:
          - name: env-json
            mountPath: /www/config/env.json
            subPath: env.json
      volumes:
        - name: env-json
          configMap:
            name: basicapp-configmap
```
{% endraw %}

## Проверка

Все изменения попали в git и можно выполнить деплой приложения командой [`werf converge`]({{ site.url }}/documentation/reference/cli/werf_converge.html):

```shell
werf converge --repo <имя пользователя Docker Hub>/werf-guided-rails
```

В результате при открытии example.com должна появиться следующая страница:

{% asset guides/rails/assets.jpg %}

Название окружения и ссылка должны соответствовать указанным в `values.yaml` настройкам.
Список лейблов должен совпадать с реальным состоянием приложения.

Под капотом это выглядит следующим образом:

{% plantuml %}
actor client
skinparam responseMessageBelowArrow true

client   -> frontend: /index.html
frontend -> backend: /index.html
backend  -> client: /index.html
client   -> frontend: <assets>
client   <- frontend: <assets>
client   -> frontend: /config/env.json
frontend -> frontend: /config/env.json
frontend -> client: /config/env.json
client   -> frontend: /api/labels
frontend -> backend: /api/labels
backend  -> client: /api/labels
{% endplantuml %}

Для того чтобы убедиться в том, что это так, посмотрим логи NGINX.

Первым шагом получим имя pod:

```shell
kubectl get pod
```

В ответ отобразится следующее:

```shell
NAME                        READY   STATUS    RESTARTS   AGE
basicapp-75dcd856b7-n8rzj   2/2     Running   0          7m34s
mysql-0                     1/1     Running   0          7m44s    
```

Далее начнём отслеживать логи контейнера `frontend` и обновим страницу без кеша (`Shift + F5`):

```shell
kubectl logs basicapp-75dcd856b7-n8rzj --container=frontend -f
```
В ответ отобразится следующее:

```shell
172.17.0.2 - - [21/Jun/2021:12:47:04 +0000] "GET / HTTP/1.1" 200 731 "-" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36"
172.17.0.2 - - [21/Jun/2021:12:47:04 +0000] "GET /packs/css/styles/labels-0b39ceb9.css HTTP/1.1" 200 96 "http://example.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36"
172.17.0.2 - - [21/Jun/2021:12:47:04 +0000] "GET /packs/js/labels-3c7370edf57a093de364.js HTTP/1.1" 200 1957 "http://example.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36"
172.17.0.2 - - [21/Jun/2021:12:47:04 +0000] "GET /config/env.json HTTP/1.1" 200 53 "http://example.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36"
172.17.0.2 - - [21/Jun/2021:12:47:04 +0000] "GET /api/labels HTTP/1.1" 200 12 "http://example.com/" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/91.0.4472.114 Safari/537.36"
```

Все запросы проходят через NGINX.

В завершении проверим, что `backend` принимает и обрабатывает `/` и `/api/labels`:
```shell
kubectl exec basicapp-75dcd856b7-n8rzj --container=backend -- tail -f log/production.log -n 11
```

В ответ отобразится следующее:

```shell
I, [2021-06-21T12:47:04.462631 #11]  INFO -- : [791ddff7037c4cd5d1dd09200bef5486] Started GET "/" for 127.0.0.1 at 2021-06-21 12:47:04 +0000
I, [2021-06-21T12:47:04.463904 #11]  INFO -- : [791ddff7037c4cd5d1dd09200bef5486] Processing by LabelsController#index as HTML
I, [2021-06-21T12:47:04.465452 #11]  INFO -- : [791ddff7037c4cd5d1dd09200bef5486]   Rendering labels/index.html.erb
I, [2021-06-21T12:47:04.466285 #11]  INFO -- : [791ddff7037c4cd5d1dd09200bef5486]   Rendered labels/index.html.erb (Duration: 0.5ms | Allocations: 131)
I, [2021-06-21T12:47:04.466685 #11]  INFO -- : [791ddff7037c4cd5d1dd09200bef5486] Completed 200 OK in 2ms (Views: 1.6ms | ActiveRecord: 0.0ms | Allocations: 422)
I, [2021-06-21T12:47:04.741920 #11]  INFO -- : [ce675af44e825d94b4f968f57778e39b] Started GET "/api/labels" for 127.0.0.1 at 2021-06-21 12:47:04 +0000
I, [2021-06-21T12:47:04.743126 #11]  INFO -- : [ce675af44e825d94b4f968f57778e39b] Processing by Api::LabelsController#index as JSON
I, [2021-06-21T12:47:04.744044 #11]  INFO -- : [ce675af44e825d94b4f968f57778e39b]   Rendering api/labels/index.json.jbuilder
D, [2021-06-21T12:47:04.745975 #11] DEBUG -- : [ce675af44e825d94b4f968f57778e39b]   Label Load (0.7ms)  SELECT `labels`.* FROM `labels`
I, [2021-06-21T12:47:04.746382 #11]  INFO -- : [ce675af44e825d94b4f968f57778e39b]   Rendered api/labels/index.json.jbuilder (Duration: 2.2ms | Allocations: 154)
I, [2021-06-21T12:47:04.746733 #11]  INFO -- : [ce675af44e825d94b4f968f57778e39b] Completed 200 OK in 3ms (Views: 2.2ms | ActiveRecord: 0.7ms | Allocations: 353)
```

Лог `backend` так же соответствует ожиданиям — цель достигнута.