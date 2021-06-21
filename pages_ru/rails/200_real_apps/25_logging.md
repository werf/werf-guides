---
title: Логирование
permalink: rails/200_real_apps/25_logging.html
---

В этой статье мы:
- рассмотрим тему логирования в приложении и просмотра логов запущенного приложения;
- возьмём за основу приложение из раздела basic-apps и добавим в него новый метод api;
- сконфигурируем rails-приложение правильным образом, чтобы писать логи, которые подхватит kubernetes.

Перейдём к состоянию приложения, в котором добавлен новый метод API:

```shell
cd werf-guides/examples/rails/020_logging/
git init --separate-git-dir ~/werf-guides-repo
```

Рассмотрим подробнее как устроено логирование в данном примере.

## Конфигурация логера и окружения

По умолчанию rails-приложение предоставляет несколько окружений: development, test и production.
В окружении production при установке переменной окружения `RAILS_LOG_TO_STDOUT=1` стандартный логер будет автоматом писать в stdout.
Поэтому стандартный логер рельсов для всех окружений переделан на такой логер, который будет писать в stdout и stderr — это наиболее правильный способ логирования приложений, запускаемых в кубах.
  - Дальнейший сбор и хранение логов, при необходимости, будет осуществляться отдельными решениями.
  - Принципиальный момент в том, что в приложении мы об этом не думаем и просто пишем в stdout/stderr.
  - Конфигурацию логера приложения для всех окружений сделана в файле (application.rb)[https://github.com/werf/werf-guides/blob/master/examples/rails/020_logging/config/application.rb]
  - Стандартная конфигурация логера для production-окружения удалена из файла (production.rb)[https://github.com/werf/werf-guides/blob/master/examples/rails/020_logging/config/environments/production.rb]

## Логирование в приложении

В наше приложение был добавлен новый метод POST /api/generate-image.
  - На вход передаётся параметр text.
  - На выходе метод генерирует png-картинку с указанным текстом.
  - В методе используется стандартный rails-логер через конструкцию `logger.debug`.
  - Полный листинг метода смотрите в (images_controller.rb)[https://github.com/werf/werf-guides/blob/master/examples/rails/100_logging/app/controllers/api/images_controller.rb]

## Деплоим приложение

Мы готовы задеплоить новую версию приложения. Сделаем коммит изменения:

```shell
git add .
git commit -m go
```

Запустим деплой:

```shell
werf converge --repo REPO
```

Проверим результат:
    
```
curl -v -X POST "example.com/api/generate-image?text=do%20it" -o /tmp/out
```

- В файле /tmp/out должно появится изображение с указанным тестом.
- Если вдруг не получилось — не беда, сейчас будем смотреть логи приложения, также в случае ошибки в файле /tmp/out будет текст ошибки.

## Как посмотреть логи

За api нашего приложения отвечает backend, который запущен в Deployment.
  - Задача deployment — создать указанное количество реплик Pod'ов.
  - Каждый Pod — это запущенный экземпляр нашего приложения на рельсах.
  - Соответственно, чтобы увидеть логи backend-а, надо запросить эти логи для одного из созданных подов.

Получим список подов:

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

Выберем один из подов с префиксом `backend` и посмотрим его логи следующей командой:

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