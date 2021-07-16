---
title: Конфигурирование инфраструктуры в виде кода
permalink: golang/100_basic/50_iac.html
layout: development
---

{% filesused title="Файлы, упомянутые в главе" %}
- .helm/templates/deployment.yaml
- .helm/templates/ingress.yaml
- .helm/templates/service.yaml
- .helm/values.yaml
{% endfilesused %}

Ранее мы уже описали инфраструктуру в виде объектов Kubernetes. Шаблонизируем эту конфигурацию с помощью [Helm](https://helm.sh/), его движок встроен в werf. Помимо этого, werf предоставляет возможности работы с секретными значениями, а также дополнительные Go-шаблоны для интеграции с собранными образами.

В этой главе мы научимся описывать Helm-шаблоны, используя возможности werf, а также освоим встроенные инструменты отладки. **Эта глава в значительной мере теоретическая, но знания, которые здесь даны, пригодятся в реальной практике.**

{% offtopic title="Что делать, если вы не работали с Helm?" %}

Не будем вдаваться в подробности [разработки YAML-манифестов с помощью Helm для Kubernetes](https://habr.com/ru/company/flant/blog/423239/). Если у вас есть вопросы о том, как именно описываются объекты Kubernetes, советуем посетить страницы документации Kubernetes о [концепциях](https://kubernetes.io/ru/docs/concepts/) и документацию Helm по [разработке шаблонов](https://helm.sh/docs/chart_template_guide/).

В первое время работа с Helm и конфигурацией для Kubernetes может быть очень сложной из-за нелепых мелочей вроде опечаток и пропущенных пробелов. Если вы только начали осваивать эти технологии — постарайтесь найти наставника, который поможет преодолеть эти сложности и посмотрит на ваши исходники сторонним взглядом.

В случае затруднений убедитесь, что вы:

- понимаете, как работает [indent](https://helm.sh/docs/chart_template_guide/function_list/#indent);
- понимаете, что такое [конструкция tuple](https://helm.sh/docs/chart_template_guide/control_structures/);
- понимаете, как Helm работает с хэш-массивами;
- очень внимательно следите за пробелами в YAML.

{% endofftopic %}

## Подстановка переменных

В описании Deployment, Ingress и Service используется значение `basicapp` — лучше заменить его переменной. Воспользуемся для этого переменной с именем чарта `.Chart.Name`. Например, было:

{% snippetcut name=".helm/templates/deployment.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/golang/020_optimize_build/.helm/templates/deployment.yaml" %}
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
```
{% endraw %}
{% endsnippetcut %}

Стало:

{% snippetcut name=".helm/templates/deployment.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/golang/025_iac/.helm/templates/deployment.yaml" %}
{% raw %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Chart.Name }}
spec:
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
```
{% endraw %}
{% endsnippetcut %}

## Конфигурирование шаблона

В реальной практике одно и то же приложение может выкатываться на различные стенды (production, test, staging и т.п.). И для разных стендов зачастую необходимо использовать разные значения.

Основной способ, который мы рекомендуем для конфигурирования — CLI-параметр ` --env`, который можно указать при вызове `converge`, например:

```shell
werf converge --repo registry.example.com/werf-guided-golang --env production
```

Передаваемый параметр, который указывает на стенд, будет доступна:

- в `werf.yaml` через конструкцию {% raw %}`{{ .Env }}`{% endraw %}
- в `.helm/templates` через конструкцию `.Values.werf.env`

{% offtopic title="Стенды и интеграция с CI" %}
Позже, в главе "Работа с инфраструктурой", мы рассмотрим, как значение этой переменной автоматически подставляется CI-системой.
{% endofftopic %}

Этот способ как правило сочетается с 

- Подстановкой значений из `values`-файлов
- Подстановкой секретных значений из `secret-values`-файлов

{% offtopic title="А как бы передать значения атрибутов через CLI?" %}
Такая опция есть в werf 1.2, но настоятельно не рекомендуется. Хочется минимизировать количество параметров передаваемых опциями командной строки, ограничившись `--env`. Такая настройка гарантирует простоту _воспроизводимости конфигурации_ на разных машинах. Окружения обычно заданы статически и выбирая конкретное окружение пользователь получает весь необходимый набор параметров для запуска приложения в данном окружении. Исходя из таких соображений нужно правильно организовывать `values` и `secret-values файлы` (см. далее) и по-возможности избегать использования опций `--set`/`--set-string`/`--set-file`/`--values`/`--secret-values`.
{% endofftopic %}

### Использование values-файлов

werf по аналогии с Helm позволяет использовать файл `/.helm/values.yaml`. Также можно указать путь до этого файла с помощью CLI-атрибута `--values PATH` по аналогии с Helm.

Создадим файл `values.yaml` и будем там хранить имя файла базы данных и скорректируем объект Deployment. Острой необходимости в этом нет, но это важный инструмент, без которого невозможна разработка реальных приложений.

{% snippetcut name=".helm/values.yaml" url="#" %}
{% raw %}
```yaml
app:
  sqlite_file:
    _default: "app.db"
    production: "app.db"
    testing: "app.db"
```
{% endraw %}
{% endsnippetcut %}

Нужные значения подставляются в helm-шаблоны:

{% snippetcut name=".helm/templates/deployment.yaml" url="#" %}
{% raw %}
```yaml
        env:
        - name: "SQLITE_FILE"
          value: {{ pluck .Values.werf.env .Values.app.sqlite_file | first | default .Values.app.sqlite_file._default | quote }}
```
{% endraw %}
{% endsnippetcut %}

Конечное значение определяется на основании переменной `.Values.werf.env`.

### Проброс значений через --set

Второй вариант подразумевает **задание переменных через CLI**. Например, в `converge` можно передать нужное значение 

```shell
werf converge --repo registry.example.com/werf-guided-golang --set "global.myvariable=somevalue"
```

И можно будет использовать это значение в шаблонах:

{% snippetcut name=".helm/templates/deployment.yaml" url="#" %}
{% raw %}
```yaml
        env:
        - name: "SQLITE_FILE"
          value: {{ .Values.global.myvariable | quote }}
```
{% endraw %}
{% endsnippetcut %}

### Использование secret-values-файлов 

<a name="secret-values-yaml" />По аналогии с файлом `/.helm/values.yaml` (или другого, заданного CLI-атрибутом `--secret-values PATH`) werf позволяет работать с секретными значениями, например, учётных данных аутентификации для сторонних сервисов, API-ключей и т.п.

Так как werf рассматривает Git как единственный источник правды, правильно хранить секретные переменные там же. Чтобы делать это корректно, мы [храним данные в шифрованном виде]({{ site.url }}/documentation/advanced/helm/working_with_secrets.html). Подстановка значений из этого файла происходит при рендере шаблона, который также запускается при деплое.

Чтобы воспользоваться секретными переменными:

* [сгенерируйте ключ]({{ site.url }}/documentation/reference/cli/werf_helm_secret_generate_secret_key.html) (`werf helm secret generate-secret-key`);
* определите ключ в текущей сессии консоли (например, `export WERF_SECRET_KEY=634f76ead513e5959d0e03a992372b8e`) или создайте файл `.werf_secret_key` в корне проекта (**этот ключ не должен храниться в Git**).

{% asset guides/common/100_50_werf_secret_key_in_gitlab.png %}

После этого можно задать секретные переменные `access_key` и `secret_key`, например, для работы с S3. Зайдите в режим редактирования секретных значений:

```shell
$ werf helm secret values edit .helm/secret-values.yaml
```

Откроется консольный текстовый редактор с данными в расшифрованном виде:

{% snippetcut name=".helm/secret-values.yaml в расшифрованном виде" url="#" ignore-tests %}
```yaml
app:
  s3:
    access_key:
      _default: bNGXXCF1GF
    secret_key:
      _default: zpThy4kGeqMNSuF2gyw48cOKJMvZqtrTswAQ
```
{% endsnippetcut %}

После сохранения значения в файле зашифруются и примут примерно такой вид:

{% snippetcut name=".helm/secret-values.yaml в зашифрованном виде" url="#" ignore-tests %}
```yaml
app:
  s3:
    access_key:
      _default: 1000f82ff86a5d766b9895b276032928c7e4ff2eeb20cab05f013e5fe61d21301427
    secret_key:
      _default: 1000bee1b42b57e39a9cfaca7ea047de03043c45e39901b8974c5a1f275b98fd0ac2c72efbc62b06cad653ebc4195b680370dc9c04e88a8182a874db286d8360def6
```
{% endsnippetcut %}

### Другие способы хранить секретные данные

Мы выбрали самый быстрый и простой способ того, как можно более-менее безопасно хранить данные в репозитории и передавать их в приложение. Но важно помнить, что любой человек с доступом к приложению в кластере (в особенности, к самой сущности Pod'а) сможет получить любой пароль.

Можно избежать этого, не выдавая доступы на исполнение команд внутри контейнеров кому-либо. Кроме того, можно собирать свои собственные образы с нуля, убирая из них небезопасные команды, а переменные помещать в [сущность Secret](https://kubernetes.io/docs/concepts/configuration/secret/#using-secrets-as-environment-variables), после чего запрещать доступ к ним всем кроме доверенных лиц.

Также можно использовать инструменты, созданные специально для таких задач — например, [Vault](https://www.vaultproject.io/). Интегрировав клиент Vault прямо в ваше приложение, можно получать любые настройки непосредственно при запуске.

Однако, все эти способы выходят за рамки самоучителя и предлагаются к самостоятельному изучению после освоения базовых приёмов.

