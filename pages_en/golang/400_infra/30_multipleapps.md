---
title: Несколько приложений в одном репозитории
permalink: golang/400_infra/30_multipleapps.html
layout: development
---


{% filesused title="Файлы, упомянутые в главе" %}
- .helm/templates/ingress.yaml
- .helm/templates/deployment-py.yaml
- .helm/templates/service-py.yaml
- werf.yaml
{% endfilesused %}

В этой главе мы добавим к нашему базовому приложению ещё одно, находящееся в том же репозитории. Это корректная ситуация для:

* сложных случаев с двумя приложениями на двух разных языках;
* ситуации, когда есть более одного запускаемого процесса (например, сервис, отвечающий на HTTP-запросы, и worker);
* ситуации, когда в одном репозитории находятся frontend и backend.

Рекомендуем также посмотреть [доклад Дмитрия Столярова](https://www.youtube.com/watch?v=g9cgppj0gKQ) о том, почему и в каких ситуациях это хороший путь для микросервисов. Также вы можете посмотреть [аналогичную статью]({{ site.url }}/documentation/guides/advanced_build/multi_images.html) о приложении с несколькими образами.

Добавим к нашему приложению простейшее Python-приложение. Подход будет очень похож на то, что делалось в главе «[Генерируем и раздаем ассеты](040_assets.html)», но с одним существенным отличием: изменения в коде Python сильно отделены от изменений в Node.js-приложении. Как следствие, мы разнесём их в разные директории, а также в различные Pod'ы.

Рассмотрим вопрос организации структуры файлов и директорий, соберём два образа (для Node.js-приложения и для Python-приложения), сконфигурируем запуск этих образов в Kubernetes.

## Структура файлов и директорий

Структура каталогов будет организована следующим образом:

```
├── .helm/
│   ├── templates/
│   └── values.yaml
├── backend/
├── bot/
└── werf.yaml
```

Для одного репозитория рекомендуется использовать один файл `werf.yaml` и одну директорию `.helm` с конфигурацией инфраструктуры. Такой подход делает работу над кодом прозрачнее и помогает избегать рассинхронизации в двух частях одного проекта.

{% offtopic title="А если получится слишком много информации в одном месте и станет сложно ориентироваться?" %}
Helm обрабатывает все файлы, которые находятся в каталоге `.helm/templates`, а значит, их может быть столько, сколько удобно вам. Для упрощения кода можно использовать [общие блоки](https://helm.sh/docs/chart_template_guide/named_templates/).

Кроме того, `werf.yaml` поддерживает [описание конфигурации в нескольких файлах]({{ site.url }}/documentation/configuration/introduction.html#%D0%BE%D0%BF%D0%B8%D1%81%D0%B0%D0%BD%D0%B8%D0%B5-%D0%BA%D0%BE%D0%BD%D1%84%D0%B8%D0%B3%D1%83%D1%80%D0%B0%D1%86%D0%B8%D0%B8-%D0%B2-%D0%BD%D0%B5%D1%81%D0%BA%D0%BE%D0%BB%D1%8C%D0%BA%D0%B8%D1%85-%D1%84%D0%B0%D0%B9%D0%BB%D0%B0%D1%85) и вынесение части кода в общие блоки.
{% endofftopic %}

## Сборка приложений

На стадии сборки приложения необходимо правильно организовать структуру файла `werf.yaml`, описав в нём сборку двух приложений на разном стеке.

Мы соберём два образа: `node` c Node.js-приложением и `bot` c Python-приложением.

{% offtopic title="Как конкретно?" %}

Сборка образа `node` аналогична ранее описанному [базовому приложению](020_basic.html) с [зависимостями](030_dependencies.html) — за исключением того, откуда берётся исходный код:

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/110-multipleapps/werf.yaml" %}
{% raw %}
```yaml
git:
- add: /nodejs
  to: /app
```
{% endraw %}
{% endsnippetcut %}

Добавляем в собираемый образ только исходные коды, относящиеся к Python-приложению. Таким образом, пересборка этой части проекта не будет срабатывать, когда изменился только Node.js-код.

Сборка для Python-приложения описана в файле `werf.yaml` как отдельный образ.

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/110-multipleapps/werf.yaml" %}
{% raw %}
```yaml
image: python
from: python:3.8-slim
git:
- add: '/python'
  to: '/app'
  stageDependencies:
    install:
    - python/requirements.txt
ansible:
  beforeInstall:
  - name: Install dependencies
    apt:
      name:
      - python3-dev
      update_cache: yes
  install:
  - name: Install pip
    pip:
      executable: pip3
      requirements: /app/requirements.txt
---
```
{% endraw %}
{% endsnippetcut %}

{% endofftopic %}

## Конфигурация инфраструктуры в Kubernetes

Подготовленные приложения будем запускать отдельными объектами Deployment: тогда в случае изменений только в одной из частей приложения будет перевыкатываться только эта часть. Создадим два отдельных файла для описания объектов: `deployment.yaml` и `deployment-py.yaml`. В условиях, когда в одном сервисе меньше 15-20 объектов, удобно следовать принципу максимальной атомарности в шаблонах.

При деплое нескольких Deployment крайне важно правильно прописать `selector`'ы в Service и Deployment:

{% snippetcut name=".helm/templates/service-py.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/110-multipleapps/.helm/templates/service-py.yaml" %}
{% raw %}
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: {{ .Chart.Name }}-py
spec:
  selector:
    app: {{ .Chart.Name }}-py
  ports:
  - name: http
    port: 5000
    protocol: TCP
```
{% endraw %}
{% endsnippetcut %}

{% snippetcut name=".helm/templates/deployment-py.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/110-multipleapps/.helm/templates/deployment-py.yaml" %}
{% raw %}
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}-py
spec:
  selector:
    matchLabels:
      app: {{ .Chart.Name }}-py
<...>
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}-py
    spec:
<...>
      containers:
      - name: python
        command: ["python","/app/app.py"]
{{ tuple "python" . | include "werf_container_image" | indent 8 }}
        workingDir: /app
        ports:
        - containerPort: 5000
          protocol: TCP
        env:
        - name: "DEBUG"
          value: "{{ pluck .Values.global.env .Values.app.isDebug | first | default .Values.app.isDebug._default }}"
{{ tuple "python" . | include "werf_container_env" | indent 8 }}
```
{% endraw %}
{% endsnippetcut %}

Маршрутизация запросов будет осуществляться через Ingress:

{% snippetcut name=".helm/templates/ingress.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/110-multipleapps/.helm/templates/ingress.yaml" %}
{% raw %}
```yaml
spec:
  rules:
  - host: {{ .Values.global.ci_url }}
    http:
      paths:
<...>
      - path: /pyapp
        backend:
          serviceName: {{ .Chart.Name }}-py
          servicePort: 5000
```
{% endraw %}
{% endsnippetcut %}

