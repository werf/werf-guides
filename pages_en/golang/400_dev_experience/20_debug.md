---
title: Упрощение отладки сборки
permalink: golang/400_dev_experience/20_debug.html
layout: development
---

{% filesused title="Файлы, упомянутые в главе" %}
- Dockerfile (удалён)
- werf.yaml
{% endfilesused %}

Воспользуемся ключевыми механизмами werf, позволяющими ускорить сборку: не будем тратить время на пересборку инфраструктуры и зависимостей, если они не изменились.

## Собираем только то, что нужно

В процессе разработки изменения вносятся в файлы с неравномерной регулярностью. Код окружения, список зависимостей и код самого приложения мы меняем с разной частотой, поэтому не хочется тратить время на перерасчёт той "стадии", что не изменилась.

Для этого в werf есть механизм [стадий сборки]({{ site.url }}/documentation/internals/stages_and_storage.html).

{% offtopic title="Что за стадии?" %}
werf подразумевает, что лучшей практикой будет разделить сборочный процесс на этапы, у каждого из которых есть свои четкие функции и назначение. Каждый такой этап соответствует промежуточному образу — подобно слоям в Docker. В werf такой этап называется стадией, и конечный образ в итоге состоит из набора собранных стадий.

Стадии — это этапы сборочного процесса, кирпичи, из которых в итоге собирается конечный образ. Стадия собирается из группы сборочных инструкций, указанных в конфигурации. Причем группировка этих инструкций не случайна, имеет определенную логику и учитывает условия и правила сборки. С каждой стадией связан конкретный Docker-образ.

Подробнее о том, какие стадии для чего предполагаются, можно посмотреть [здесь]({{ site.url }}/documentation/internals/stages_and_storage.html). Если вкратце, то werf предлагает использовать для стадий следующую стратегию:

*   использовать стадию `beforeInstall` для инсталляции системных пакетов;
*   `install` — для инсталляции системных зависимостей и зависимостей приложения;
*   `beforeSetup` — для настройки системных параметров и установки приложения;
*   `setup` — для настройки приложения.

Другие подробности о стадиях описаны в [документации]({{ site.url }}/documentation/configuration/stapel_image/assembly_instructions.html).

Одно из основных преимуществ использования стадий в том, что мы можем перезапускать сборку не с нуля, а только с той стадии, которая зависит от изменений в определенных файлах.
{% endofftopic %}

Для работы механизма стадий нам нужно будет перенести код сборки из `Dockerfile` в `werf.yaml`, скорректировав синтаксис.

Сейчас в `werf.yaml` сборка описана так:

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/nodejs/015_deploy_app/werf.yaml" %}
{% raw %}
```yaml
---
image: basicapp
dockerfile: Dockerfile
```
{% endraw %}
{% endsnippetcut %}

Удалите эти строки — мы перепишем их, прописав там:

- базовый образ (в `Dockerfile` это было `FROM node:14-stretch`)
- рабочую папку (в `Dockerfile` это было `WORKDIR /app`)
- сценарий добавления в финальный образ исходного кода (в `Dockerfile` это было `COPY . .`)
- установку зависимостей

В последнем, к слову важно, что работа с `apt` и `npm` должна быть реализована в правильных стадиях. Причём важно сказать werf-и, что при изменении файла `package.json` нужно [перезапускать стадию]({{ site.url }}/documentation/advanced/building_images_with_stapel/assembly_instructions.html#%D0%B7%D0%B0%D0%B2%D0%B8%D1%81%D0%B8%D0%BC%D0%BE%D1%81%D1%82%D1%8C-%D0%BE%D1%82-%D0%B8%D0%B7%D0%BC%D0%B5%D0%BD%D0%B5%D0%BD%D0%B8%D0%B9-%D0%B2-git-%D1%80%D0%B5%D0%BF%D0%BE%D0%B7%D0%B8%D1%82%D0%BE%D1%80%D0%B8%D0%B8) `install` (в которой будет запускаться `npm ci`).

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/nodejs/020_optimize_build/werf.yaml" %}
{% raw %}
```yaml
---
image: basicapp
from: node:14-stretch
git:
- add: /
  to: /app
  stageDependencies:
    install:
    - package.json
shell:
  beforeInstall:
  - apt update
  - apt install -y tzdata locales
  install:
  - cd /app && npm ci
docker:
  WORKDIR: /app
```
{% endraw %}
{% endsnippetcut %}

{% offtopic title="Что тут написано?" %}
Здесь используется 4 корневых директивы: `from` — на основании какого образа будет осуществляться сборка; [`git`]({{ site.url }}/documentation/advanced/building_images_with_stapel/git_directive.html) — описывает импорт данных из репозитория; [`shell`]({{ site.url }}/documentation/advanced/building_images_with_stapel/assembly_instructions.html#shell) — описывает сборку стадий с помощью shell-команд; [`docker`]({{ site.url }}/documentation/advanced/building_images_with_stapel/docker_directive.html) — инструкции для Docker.
{% endofftopic %}

Разумеется, наш пример очень простой. Реальные сценарии сборки гораздо сложнее:

- иногда нужно вытащить код приложения из другого репозитория
{% offtopic title="директива `git` это позволяет" %}
Директива `git` также позволяет добавлять код из удалённых Git-репозиториев, используя параметр `url`.

Пример:
{% raw %}
```yaml
git:
- add: /src
  to: /app
- url: https://github.com/ariya/phantomjs
  tag: 2.2.1
  add: /
  to: /src/phantomjs
```
{% endraw %}
Детали и особенности можно почитать в [документации]({{ site.url }}/documentation/configuration/stapel_image/git_directive.html#работа-с-удаленными-репозиториями).

При использовании удалённых git-репозиториев важно указывать конкретный тег или коммит, чтобы получить детерминированную сборку.
{% endofftopic %}

- убирать или добавлять код приложения по более сложным сценариям
{% offtopic title="можно добавлять и исключать файлы по маске" %}
В реальной практике нужно добавлять файлы, **фильтруя по названию или пути**.

К примеру, в данном варианте добавляются все файлы `.php` и `.js` из каталога `/src`, исключая файлы с суффиксом `-dev.` и `-test.` в имени файла.
{% raw %}
```yaml
git:
- add: /src
  to: /app
  includePaths:
  - '**/*.php'
  - '**/*.js'
  excludePaths:
  - '**/*-dev.*'
  - '**/*-test.*'
```
{% endraw %}
{% endofftopic %}

- корректировать владельцев файлов
{% offtopic title="можно указать владельцев в директиве `git`" %}
В Linux-системах важно не забывать про установку **владельца файлов**.

Например, вот так изменяется владелец файла `index.php` на `www-data`:

```yaml
git:
- add: /src/index.php
  to: /app/index.php
  owner: www-data
```
{% endofftopic %}

- использовать шаблонизацию
{% offtopic title="werf использует go templates" %}
В werf поддерживаются [**Go templates**]({{ site.url }}/documentation/configuration/introduction.html#%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD%D1%8B-go), поэтому легко определять переменные и записывать в них константы и часто используемые образы.

Например, сделаем 2 образа, используя один базовый образ `golang:1.11-alpine`:

{% raw %}
```yaml
{{ $base_image := "golang:1.11-alpine" }}

project: my-project
configVersion: 1
---

image: gogomonia
from: {{ $base_image }}
---
image: saml-authenticator
from: {{ $base_image }}
```
{% endraw %}

Подробнее почитать про Go-шаблоны в werf можно в документации: [**werf go templates**]({{ site.url }}/documentation/configuration/introduction.html#%D1%88%D0%B0%D0%B1%D0%BB%D0%BE%D0%BD%D1%8B-go).
{% endofftopic %}

## Запускаем сборку

Ваш `werf.yaml` должен был принять вид:

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/nodejs/020_optimize_build/werf.yaml" %}
{% raw %}
```yaml
project: werf-guided-golang
configVersion: 1
---
image: basicapp
from: node:14-stretch
git:
- add: /
  to: /app
  stageDependencies:
    install:
    - package.json
shell:
  beforeInstall:
  - apt update
  - apt install -y tzdata locales
  install:
  - cd /app && npm ci
docker:
  WORKDIR: /app
```
{% endraw %}
{% endsnippetcut %}

Сделайте коммит изменений в репозитории с кодом, затем запустите `converge` и обратите внимание на время сборки

```shell
werf converge --repo registry.example.com/werf-guided-golang
```

Попробуйте менять список зависимостей (просто добавьте какой-нибудь пакет в `package.json`), файл с кодом (`app.js`) или инфраструктуру (добавьте новый аттрибут с произвольным текстом в секцию `metadata:` в файле `deployment.yaml`) и посмотрите, как быстро осуществляется сборка и где используется ранее собранный кусок образа.

