---
title: Внесение изменений
permalink: rails/100_basic/35_deploy_changes.html
---
Внесём изменения в уже выкаченное приложение и его инфраструктуру. Продемонстрируем, как работает подход infrastructure-as-code (IaC).

## Работаем с инфраструктурой
### Просмотр состояния

Чтобы посмотреть состояние запущенного приложения в Kubernetes, существует несколько команд.

Получить список запущенных Deployment'ов и Pod'ов:
```shell
kubectl get deployment
kubectl get pod
```

Посмотреть summary-информацию по Deployment'у `basicapp`:
```shell
kubectl describe deployment basicapp
```

Deployment запускает Pod'ы. Логи пишутся в Pod'ах. Такой командой можно получить логи одного из запущенных Pod'ов:
```shell
kubectl logs basicapp-<podId>
```

### Масштабирование

Веб-сервер запущен в Deployment'е `basicapp`. Помотрим, сколько реплик запущено:

```shell
kubectl get pod
```

Ответ будет примерно таким:
```shell
NAME                      READY   STATUS    RESTARTS   AGE
basicapp-57789b68-kxcb9   1/1     Running   0          72m
```

Поменяем вручную на 2 реплики:
```shell
kubectl edit deployment basicapp
```

В открывшемся редакторе выставляем `spec.replicas=2`, закрываем редактор.
Снова посмотрим, сколько реплик запущено:
```shell
kubectl get pod
```

Ответ станет примерно таким:
```shell
NAME                      READY   STATUS    RESTARTS   AGE
basicapp-57789b68-c2xlq   1/1     Running   0          6s
basicapp-57789b68-kxcb9   1/1     Running   0          72m
```

Мы произвели масштабирование вручную. Теперь снова запустим `werf converge`:
```shell
werf converge ...
```

И снова посмотрим, сколько реплик запущено:
```shell
kubectl get pod
```
```shell
NAME                      READY   STATUS    RESTARTS   AGE
basicapp-57789b68-kxcb9   1/1     Running   0          72m
```

Количество реплик соответствует таковому в Git-репозитории. Дело в том, что werf привела состояние кластера к состоянию, описанному в текущем Git-коммите. Этот принцип называется **гитерминизмом** (giterminism).

Как же соблюсти **гитерминизм** и сделать всё правильно? Изменим тот же `spec.replicas`, но уже через состояние приложения, описанное в Git. 

Перейдём к новому состоянию приложения, в котором выставлен `spec.replicas=2` для Deployment'а `basicapp`:
```shell
cd werf-guides/examples/rails/016_scale/
git init --separate-git-dir ~/werf-guides-repo
```

Закоммитим изменения:
```shell
git add .
git commit -m go
```

Запустим деплой:
```shell
werf converge --repo REPO
```

## Меняем приложение

Наше приложение ­— это [CRUD](https://ru.wikipedia.org/wiki/CRUD) для создания labels. Ниже рассмотрим некоторые действия, которые можно выполнить с приложением.

### Работа с приложением

Продемонстрируем получение списка labels из консоли:
```shell
curl "http://URL:PORT/api/labels"
```

Создадим новые labels из консоли:
```shell
curl -X POST "http://URL:PORT/api/labels/?label=red"
curl -X POST "http://URL:PORT/api/labels/?label=hot"
curl -X POST "http://URL:PORT/api/labels/?label=blue"
curl -X POST "http://URL:PORT/api/labels/?label=cold"
```

Получим обновлённый список labels:
```shell
curl "http://URL:PORT/api/labels"
```

### Добавление полей

Добавим новые timestamp-поля для label: время его создания и обновления.

Перейдём к новому состоянию приложения, в котором добавлены новые поля (`created_at` и `updated_at` в таблице `labels`):
```shell
cd werf-guides/examples/rails/017_add_fields
git init --separate-git-dir ~/werf-guides-repo
```

Внесённые изменения включают в себя:
  - новый файл миграций [20210526202700_add_timestamps_to_labels.rb](https://github.com/werf/werf-guides/tree/master/examples/rails/017_add_fields/db/migrate/20210526202700_add_timestamps_to_labels.rb);
  - обновлённую схему БД [schema.rb](https://github.com/werf/werf-guides/tree/master/examples/rails/017_add_fields/db/schema.rb);
  - правки в [_label.json.jbuilder](https://github.com/werf/werf-guides/tree/master/examples/rails/017_add_fields/app/views/api/labels/_label.json.jbuilder), чтобы API выдавал не только `id` и имя `label`, но и поля `created_at` и `updated_at`.

Сделаем коммит изменений:
```shell
git add .
git commit -m go
```

Запустим деплой:
```shell
werf converge --repo REPO
```

Проверим результат:
```shell
curl -X POST "http://URL:PORT/api/labels/?label=black"
curl -X POST "http://URL:PORT/api/labels/?label=white"
curl "http://URL:PORT/api/labels"
```

В ответе увидим новые поля. Поздравляем, у нас получилось!

### Обеспечим корректность работы нескольких реплик

В данный момент есть проблема: наше приложение использует БД SQLite в локальном файле, а мы добавили несколько реплик. Получается, что каждая реплика использует свою базу данных, а не общую. Это приведёт к тому, что запросы будут равномерно распределяться по репликам при создании и получении labels, а результаты запросов могут отличаться при перезапуске. Чтобы решить эту проблему, давайте переведем наше приложение на общую БД — MySQL.

Перейдём к новому состоянию приложения, в котором оно переключено на MySQL:

```shell
cd werf-guides/examples/rails/018_fixup_consistency
git init --separate-git-dir ~/werf-guides-repo
```

Рассмотрим, какие изменения были внесены в приложение:
 - Создан новый [шаблон](https://github.com/werf/werf-guides/tree/master/examples/rails/018_fixup_consistency/.helm/templates/database.yml) для запуска MySQL в кластере. В реальных приложениях MySQL запустить несколько сложнее, т.к. нужен persistent volume. Но в нашем случае для development-окружения это не требуется: БД будет терять все свои данные в случае перезапуска.
 - Deployment приложения адаптирован так, чтобы миграции запускались в init-контейнере. Также настройки SQLite более не нужны: [deployment.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/018_fixup_consistency/.helm/templates/deployment.yaml).
 - Приложение настроено на работу с MySQL:
    - [database.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/018_fixup_consistency/config/database.yml)
    - [Gemfile](https://github.com/werf/werf-guides/tree/master/examples/rails/018_fixup_consistency/Gemfile)
    - [Gemfile.lock](https://github.com/werf/werf-guides/tree/master/examples/rails/018_fixup_consistency/Gemfile.lock)

Сделаем коммит изменений:
```shell
git add .
git commit -m go
```

Запустим деплой:
```shell
werf converge --repo REPO
```

Дождёмся выполнения команды. Заметьте, что в процессе работы в логах могут появляться ошибки подключения к БД, потому что контейнер с ней ещё не успел запуститься. Это нормально - необходимо дождаться полного запуска приложения.

Проверим создание и получение labels и увидим консистивное поведение:
```shell
curl -X POST "http://URL:PORT/api/labels/?label=name"
curl "http://URL:PORT/api/labels/" 
```

Так как у нас rails-приложение, мы когда-нибудь захотим зайти в работающий контейнер запущенного приложения и запустить там `rails console`. Это можно сделать вот так _(в общем случае это плохая практика, но допустимо на данном этапе для лучшей наглядности)_:
```shell
kubectl exec -ti basicapp-57789b68-c2xlq -- bash
rails c
```

Получим список labels напрямую из базы данных:
```shell
Label.all
```