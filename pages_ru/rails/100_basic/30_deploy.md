---
title: Деплой приложения
permalink: rails/100_basic/30_deploy.html
---

В предыдущих главах мы собрали образ приложения и подготовили окружение для его развертывания. Теперь развернём приложение в ранее подготовленном кластере Kubernetes.

При деплое в Kubernetes используются Kubernetes-манифесты, которые описывают ресурсы (объекты Kubernetes), необходимые для работы приложений. Эти ресурсы включают в себя, к примеру, Deployment, отвечающий за запуск приложений в контейнерах, и Service/Ingress, отвечающие за доступ к запущенным приложениям изнутри и извне кластера.

## Deployment

Ресурс Deployment создаёт набор ресурсов, запускающих приложение. Создадим для него файл `.helm/templates/deployment.yaml` с таким содержимым:
{% snippetcut name=".helm/templates/deployment.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/rails/015_deploy_app/.helm/templates/deployment.yaml" %}
{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: basicapp
spec:
  selector:
    matchLabels:
      app: basicapp
  template:
    metadata:
      labels:
        app: basicapp
    spec:
      imagePullSecrets:
      - name: registrysecret  # Имя ресурса Secret с логином и паролем для Docker Hub.
      containers:
      - name: basicapp
        command: ["/bin/bash", "-ec", "bundle exec rails db:migrate && bundle exec puma"]
        image: {{ .Values.werf.image.basicapp }}
        ports:
        - containerPort: 3000
        env:
        - name: "SQLITE_FILE"
          value: "app.db"
```
{% endraw %}
{% endsnippetcut %}

Здесь мы создали шаблон werf для создания ресурса Deployment. Этот шаблон по сути является Helm-шаблоном, но с некоторой [дополнительной функциональностью]({{ site.url }}/documentation/v1.2/advanced/helm/overview.html), которую предлагает werf. Например, конструкция {% raw %}`image: {{ .Values.werf.image.basicapp }}`{% endraw %} здесь используется для того, чтобы werf автоматически подставлял генерируемый тег образа (и имя образа) в поле `image`, так как тег образа становится известен только во время сборки.

Werf пересобирает образы только при изменениях в файлах, указанных в `werf.yaml`, а также при изменении самого `werf.yaml`. При пересборке изменится и тег образа, что приведёт к обновлению Deployment'а. Если же изменений в вышеупомянутых файлах нет, то образ не пересоберётся, а Deployment и создаваемые им ресурсы не передплоятся, т.к. в этом просто нет необходимости: в кластере уже самая свежая версия приложения.

## Service

Ресурс Service позволяет другим приложениям в кластере обращаться к нашему приложению. Создадим для него файл `.helm/templates/service.yaml` с таким содержимым:
{% snippetcut name=".helm/templates/service.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/rails/015_deploy_app/.helm/templates/service.yaml" %}
{% raw %}
```yaml
apiVersion: v1
kind: Service
metadata:
  name: basicapp
spec:
  selector:
    app: basicapp
  ports:
  - name: http
    port: 3000
```
{% endraw %}
{% endsnippetcut %}

## Ingress

Ресурс Ingress позволяет открыть доступ к нашему приложению *снаружи* кластера (в отличие от Service, который разрешает доступ только между приложениями *внутри* кластера). В Ingress'е мы указываем, на какой Service должен пойти внешний трафик, который попадает на домен `example.com`. Создадим для Ingress'а файл `.helm/templates/ingress.yaml` с таким содержимым:

{% snippetcut name=".helm/templates/ingress.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/rails/015_deploy_app/.helm/templates/ingress.yaml" %}
{% raw %}
```yaml
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
          servicePort: 3000
```
{% endraw %}
{% endsnippetcut %}

Также добавим наш домен в конфигурацию Rails (`config/application.rb`):
{% snippetcut name="config/application.rb" url="#" %}
{% raw %}
```ruby
...
module DemoApplication
  class Application < Rails::Application
    config.hosts << 'example.com'
    ...
```
{% endraw %}
{% endsnippetcut %}

## Деплой в Kubernetes

Сохраним изменения перед сборкой/деплоем:
```bash
git add .helm config
git commit -m "Add deploy configuration"
```

Команда [werf converge]({{ site.url }}/documentation/reference/cli/werf_converge.html) выполнит сразу и сборку, и развертывание приложения в Kubernetes:
```bash
werf converge --repo <имя пользователя Docker Hub>/werf-guided-rails
```

Результат выполнения в случае успешной сборки и деплоя:
```bash
...
│ basicapp/dockerfile  Successfully built 4c1054085159
│ │ basicapp/dockerfile  Successfully tagged 93c05bf8-c459-4768-b388-3cdbc80e2868:latest
│ ├ Info
│ │       name: .../werf-guided-rails:f4caaa836701e5346c4a0514bb977362ba5fe4ae114d0176f6a6c8cc-1612277803607
│ │       size: 371.4 MiB
│ └ Building stage basicapp/dockerfile (40.31 seconds)
└ :boat: image basicapp (41.13 seconds)
...
┌ Waiting for release resources to become ready
│ ┌ Status progress
│ │ DEPLOYMENT                                                                                                                                                      REPLICAS                      AVAILABLE                        UP-TO-DATE
│ │ basicapp                                                                                                                                                        1/1                           1                                1
│ │ │   POD                                                           READY                  RESTARTS                       STATUS
│ │ └── 687f8cc569-n6gkw                                              1/1                    0                              Running
│ └ Status progress
└ Waiting for release resources to become ready (0.02 seconds)

NAME: werf-guided-rails
LAST DEPLOYED: Tue Feb  2 21:57:23 2021
NAMESPACE: werf-guided-rails
STATUS: deployed
REVISION: 1
TEST SUITE: None
Running time 62.66 seconds
```

Теперь приложение доступно по ссылке [http://example.com/api/labels](http://example.com/api/labels).
