---
title: Динамические окружения
permalink: golang/400_infra/40_dynamicenvs.html
layout: development
---

{% filesused title="Файлы, упомянутые в главе" %}
- .gitlab-ci.yml
{% endfilesused %}

В этой главе мы добавим к нашему базовому приложению возможность выкатываться на множество окружений, организуя так называемые «feature-ветки» или «review-окружения».

Нередко необходимо разрабатывать и тестировать сразу несколько features для вашего приложения и нет понимания, как это делать, когда у вас всего два окружения. Разработчику или тестеру приходится дожидаться своей очереди на контуре staging и затем проводить необходимые манипуляции с кодом (тестирование, отладка, демонстрация функционала). Из-за этого разработка сильно замедляется. Динамические окружения позволяют развернуть и погасить окружение под каждую ветку в любой момент, разблокируя процессы тестирования.

Для того, чтобы организовать динамические окружения, необходимо, чтобы:

* Домен, на котором разворачивается приложение, конфигурировался в объекте Ingress на основании значения из `.gitlab-ci.yml` (мы сделали это в главе «[Базовое приложение](020_basic.html)»).
* В `.gitlab-ci.yml` было прописано создание и удаление review-окружений.
* Во вновь создаваемых review-окружениях должен создаваться секрет с API-ключом для доступа к registry.

Пропишем в`.gitlab-ci.yml` создание review-окружений:

{% snippetcut name=".gitlab-ci.yml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/120-dynamicenvs/.gitlab-ci.yml" %}
{% raw %}
```yaml
Deploy to Review:
  extends: .base_deploy
  stage: deploy
  environment:
    name: review/${CI_COMMIT_REF_SLUG}
    url: http://${CI_COMMIT_REF_SLUG}.example.com
    on_stop: Stop Review
  only:
    - /^feature-*/
  when: manual
```
{% endraw %}
{% endsnippetcut %}

Создаваемые окружения нужно не забывать отключать: в противном случае ресурсы в кластере закончатся. Мы добавили зависимость `on_stop: Stop Review`, которая означает, что мы будем останавливать наше окружение стадией `Stop Review`. Сама стадия описывается следующим образом:

{% snippetcut name=".gitlab-ci.yml" url="https://github.com/werf/werf-guides/blob/master/examples/gitlab-nodejs/120-dynamicenvs/.gitlab-ci.yml" %}
{% raw %}
```yaml
Stop Review:
  stage: deploy
  variables:
    GIT_STRATEGY: none
  script:
    - werf dismiss --env $CI_ENVIRONMENT_SLUG --namespace ${CI_ENVIRONMENT_SLUG} --with-namespace
  when: manual
  environment:
    name: review/${CI_COMMIT_REF_SLUG}
    action: stop
  only:
    - /^feature-*/
```
{% endraw %}
{% endsnippetcut %}

Удаление Helm-релиза (и, как следствие, всех выкаченных им объектов) осуществляется командой [`werf dismiss`]({{ site.url }}/documentation/cli/main/dismiss.html). Мы указываем название окружения ([`--env $CI_ENVIRONMENT_SLUG`](https://docs.gitlab.com/ee/ci/environments/#environment-variables-and-runner)).

Атрибут `GIT_STRATEGY: none` говорит runner'у, что check out кода не требуется, чем немного ускоряется операция и разгружается runner.

При таком CI-процессе можно выкатывать каждую ветку `feature/*` в отдельный namespace с изолированной базой данных, выполнять на ней необходимые миграции и, например, проводить тесты для данного окружения.

Также необходимо организовать [доступ кластера к registry](020_basic/20_iac.html#registryaccess) в каждом создаваемом окружении.


