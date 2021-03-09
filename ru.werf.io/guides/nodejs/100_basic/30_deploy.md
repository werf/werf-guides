---
title: Деплой приложения
permalink: nodejs/100_basic/30_deploy.html
---

{% filesused title="Файлы, упомянутые в главе" %}
- .helm/templates/deployment.yaml
- .helm/templates/registry-secret.yaml
- .helm/templates/ingress.yaml
- .helm/templates/service.yaml
{% endfilesused %}

В предыдущей главе мы описали IaC для сборки, теперь нужно описать IaC для запуска в Kubernetes. В нашем случае потребуются следующие объекты Kubernetes: Deployment, Service и Ingress.

{% offtopic title="Как быть, если я не знаю Kubernetes?" %}
В самоучителе будет приведён исходный код инфраструктуры и вы сможете интуитивно догадаться, что там написано. Чтобы научиться писать подобный код самостоятельно, стоит воспользоваться обучающими материалами и/или, например, в [официальной документацией Kubernetes](https://kubernetes.io/docs/tutorials/kubernetes-basics/). Видеокурсов и учебников, рассказывающих об объектах Kubernetes и их возможных настройках, на данный момент существует достаточно.
{% endofftopic %}

werf поддерживает весь функционал шаблонизатора Helm (werf использует вкомпилированный в него Helm для деплоя), а также предоставляет [дополнительную функциональность]({{ site.docsurl }}/documentation/advanced/helm/basics.html). Подробнее в шаблонизации и правилах написания Kubernetes-объектов мы разберёмся позже, в главе «Конфигурирование инфраструктуры в виде кода», а пока — добьёмся, чтобы приложение заработало в реальном кластере.

## Deployment

Объект Deployment позволяет создать объект Pod, который содержит в себе контейнеры с приложениями и управляет ими. У создаваемого нами Pod будет один контейнер — `basicapp`.

{% snippetcut name=".helm/templates/deployment.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/nodejs/015_deploy_app/.helm/templates/deployment.yaml" %}
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
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
  replicas: 1
  template:
    metadata:
      labels:
        app: basicapp
    spec:
      containers:
      - name: basicapp
        command: ["node","/app/app.js"]
        image: {{ .Values.werf.image.basicapp }}
        workingDir: /app
        ports:
        - containerPort: 3000
          protocol: TCP
        env:
        - name: "SQLITE_FILE"
          value: "app.db"
```
{% endraw %}
{% endsnippetcut %}

Обратите внимание на конструкцию {% raw %}`image: {{ .Values.werf.image.basicapp }}`{% endraw %} — с помощью неё подставляется актуальное имя образа.

werf пересобирает контейнеры только при необходимости, если есть изменения в связанных файлах в Git или в конфигурации `werf.yaml`. При изменении образа меняется его тег, что приводит к перевыкату Pod'а с новым образом.

## Registry Secret

Kubernetes-кластер для запуска приложения использует образы из registry. Поэтому важно, чтобы кластер мог авторизоваться в registry. Как правило, ситуация отличается для локального и внешнего registry.

<div style="display: flex; justify-content: space-between; margin: 0 10px 0 20px;">
<div class="button__blue button__blue_inline expand_columns_button" id="local_cluster_button"><a href="#">локальный registry</a></div>
<div class="button__blue button__blue_inline expand_columns_button" id="remote_cluster_button"><a href="#">внешний registry</a></div>
</div>

{% expandonclick id="local_cluster_button__content" %}
{% include_relative 30_deploy_registrysecret_local.md %}
{% endexpandonclick %}

{% expandonclick id="remote_cluster_button__content" %}
{% include_relative 30_deploy_registrysecret_remote.md %}
{% endexpandonclick %}

## Service

Объект Service позволяет приложениям в кластере обнаруживать друг друга. Пропишем его:

{% snippetcut name=".helm/templates/service.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/nodejs/015_deploy_app/.helm/templates/service.yaml" %}
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
    protocol: TCP
```
{% endraw %}
{% endsnippetcut %}

## Ingress

Объект Ingress позволяет организовать маршрутизацию трафика на созданный Service для нужного домена (в нашем примере — `example.com`).

{% offtopic title="Что за объект Ingress?" %}
Возможна коллизия терминов:

* Есть [NGINX Ingress Controller](https://github.com/kubernetes/ingress-nginx), который работает в кластере и принимает входящие извне запросы.
* А ещё есть [объект Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/), который фактически описывает настройки для NGINX Ingress Controller.

В статьях и бытовой речи оба этих термина зачастую называют «Ingress», так что догадываться нужно по контексту.
{% endofftopic %}

{% snippetcut name=".helm/templates/ingress.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/nodejs/015_deploy_app/.helm/templates/ingress.yaml" %}
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

## Выкат в кластер

Воспользуемся [командой `converge`]({{ site.docsurl }}/documentation/reference/cli/werf_converge.html) для того, чтобы собрать образ, загрузить собранный образ в registry и задеплоить приложение в Kubernetes. Единственной опцией указывается репозиторий для хранения образов `--repo registry.example.com/werf-guided-nodejs`.

{% offtopic title="Как будут храниться образы в репозитории?" %}
При организации деплоя без использования werf зачастую приходится формализовать принципы, по которым именуются образы в registry. В нашем случае нет необходимости думать об этом: werf берёт организацию тегирования на себя.

werf реализует тегирование, которое называется content-based (т.е. основанное на содержимом): образы меняются и выкатываются автоматически, если меняется состояние контента в Git.

Если вы хотите узнать подробности — читайте [в документации]({{ site.docsurl }}/documentation/internals/stages_and_storage.html#%D0%B8%D0%BC%D0%B5%D0%BD%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5-%D1%81%D1%82%D0%B0%D0%B4%D0%B8%D0%B9) и [статье на Хабре](https://habr.com/ru/company/flant/blog/495112/).
{% endofftopic %}

Сделайте коммит изменений в репозитории с кодом и затем выполните команду:

```shell
werf converge --repo registry.example.com/werf-guided-nodejs
```

Для локального registry без авторизации:

```shell
werf converge --repo registry.example.com:5000/werf-guided-nodejs --insecure-registry=true
```

В результате вы должны увидеть логи примерно такого вида:

```
│ │ basicapp/dockerfile  Successfully built 706dc48a845e
│ │ basicapp/dockerfile  Successfully tagged 4d3e9466-88b7-4b82-a6a9-bf24a17f6331:latest
│ ├ Info
│ │       name: registry.example.com:5000/werf-guided-nodejs:0ca86cb6b4476c704a032a2e42cc3a6a76579f63d4888907b1dddb0a-1615207112506
│ │       size: 375.4 MiB
│ └ Building stage basicapp/dockerfile (70.20 seconds)
└ ⛵ image basicapp (71.05 seconds)

Release "werf-guided-nodejs" does not exist. Installing it now.

┌ Waiting for release resources to become ready
│ ┌ Status progress
│ │ DEPLOYMENT                                                                                                                                                      REPLICAS                      AVAILABLE                        UP-TO-DATE
│ │ basicapp                                                                                                                                                        1/1                           1                                1
│ │ │   POD                                                           READY                  RESTARTS                       STATUS
│ │ └── 6cf5b444bc-6rh4j                                              1/1                    0                              Running
│ └ Status progress
└ Waiting for release resources to become ready (4.02 seconds)

NAME: werf-guided-nodejs
LAST DEPLOYED: Tue Nov 17 16:29:16 2020
NAMESPACE: werf-guided-nodejs
STATUS: deployed
REVISION: 1
TEST SUITE: None
Running time 75.87 seconds
```

После этого приложение должно быть доступно в браузере:

![](/guides/images/template/100_30_app_in_browser.png)

<div id="go-forth-button">
    <go-forth url="40_optimize.html" label="Ускорение сборки" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
