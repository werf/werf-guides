---
title: Логирование
permalink: rails/200_real_apps/25_logging.html
examples: examples/rails/100_logging
layout: wip
chapter_initial_prepare_cluster: true
chapter_initial_prepare_repo: true
description: |
    В этой статье мы добавим в наше приложение новый метод API, рассмотрим тему логирования и просмотра логов запущенного приложения, сконфигурируем приложение чтобы логи мог подхватить Kubernetes.
---

## Настройка logger и окружения

По умолчанию rails-приложение предоставляет несколько окружений: development, test и production.
В окружении production при установке переменной окружения `RAILS_LOG_TO_STDOUT=1` стандартный логер по умолчанию пишет логи в stdout, но в нашем случае он доработан и пишет еще и в stderr для всех окружений — это наиболее правильный способ логирования приложений, запускаемых в Kubernetes-кластере.

В нашем приложении сбор и хранение логов возможен при сторонних приложений и это основной принципиальный момент - мы об этом не думаем и просто пишем логи в stdout/stderr.

Конфигурация логера приложения для всех окружений сделана в файле {% include snippetcut_example path="config/application.rb" syntax="ruby" examples=page.examples %} а стандартная конфигурация логера для production-окружения удалена из файла production.rb.
{% include snippetcut_example path="config/environments/production.rb" syntax="ruby" examples=page.examples %}

## Логирование в приложении

В наше приложение был добавлен новый метод POST `/api/generate-image`, использующий стандартный rails-логер через конструкцию `logger.debug`. На вход метода передаётся параметр `text`, а на выходе генерируется png-картинка с переданным `text`.

Полный листинг метода можно посмотреть в файле images_controller.rb.
{% include snippetcut_example path="app/controllers/api/images_controller.rb" syntax="ruby" examples=page.examples %}

## Деплоим приложение

Сначала сделаем коммит изменения:

```shell
git add .
git commit -m go
```

Запустим деплой:

```shell
werf converge --repo <имя пользователя Docker Hub>/werf-guided-rails
```

Проверим результат:

```shell
curl -v -X POST "example.com/api/generate-image?text=do%20it" -o /tmp/out
```

В файле /tmp/out должно появится изображение с указанным тестом, но если вдруг не получилось, то в следующем пункте мы посмотрим логи из которых можно сделать вывод о работоспособности реализованного функционала, а также в случае возникновения ошибки в файле /tmp/out будет её текст.

## Как посмотреть логи

За API нашего приложения отвечает backend, представленный в виде Deployment с определенным количеством реплик Pod'ов, где каждый Pod - это запущенный экземпляр нашего приложения, таким образом, чтобы посмотреть его логи нужно запросить их для одного из Pod'ов.

Получим список всех Pod'ов:

```shell
kubectl -n werf-guided-rails get pod
```

В ответ отобразится следующее:
```shell
NAME                      READY   STATUS    RESTARTS   AGE
backend-7964b6b68-n28v5   1/1     Running   0          21m
backend-7964b6b68-wzgw4   1/1     Running   0          21m
mysql-666f76d7cb-967xb    1/1     Running   3          4d6h
```

Выберем один из Pod'ов с префиксом `backend` и посмотрим его логи следующей командой:

```shell
kubectl -n werf-guided-rails logs -f backend-7964b6b68-n28v5
```

В ответе увидим:
```shell
I, [2021-06-07T16:47:28.032279 #1]  INFO -- : Started POST "/api/generate-image?text=do%20it" for 172.17.0.2 at 2021-06-07 16:47:28 +0000
I, [2021-06-07T16:47:28.035225 #1]  INFO -- : Processing by Api::ImagesController#generate_image as JSON
I, [2021-06-07T16:47:28.035426 #1]  INFO -- :   Parameters: {"text"=>"do it"}
D, [2021-06-07T16:47:28.035787 #1] DEBUG -- : received generate_image request
D, [2021-06-07T16:47:28.036043 #1] DEBUG -- : start generating image for text "do it"
D, [2021-06-07T16:47:28.498286 #1] DEBUG -- : finish generation image for text "do it"
I, [2021-06-07T16:47:28.498673 #1]  INFO -- :   Rendering text template
I, [2021-06-07T16:47:28.498811 #1]  INFO -- :   Rendered text template (Duration: 0.0ms | Allocations: 1)
I, [2021-06-07T16:47:28.498897 #1]  INFO -- : Sent data  (0.5ms)
I, [2021-06-07T16:47:28.498972 #1]  INFO -- : Completed 200 OK in 463ms (Views: 0.4ms | ActiveRecord: 0.0ms | Allocations: 247)
```
Ура! У нас всё получилось.
