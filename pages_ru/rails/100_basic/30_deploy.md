---
title: Деплой приложения
permalink: rails/100_basic/30_deploy.html
examples: examples/basic/002_deploy
examples_initial: examples/basic/001_build
description: |
    В предыдущих главах мы собрали образ приложения и подготовили окружение для его развертывания. Теперь развернём приложение в ранее подготовленном кластере Kubernetes.

    При деплое в Kubernetes используются Kubernetes-манифесты, которые описывают ресурсы (объекты Kubernetes), необходимые для работы приложений. Эти ресурсы включают в себя, к примеру, Deployment, отвечающий за запуск приложений в контейнерах, и Service/Ingress, отвечающие за доступ к запущенным приложениям изнутри и извне кластера.
---

## Deployment
Ресурс Deployment создаёт набор ресурсов, запускающих приложение. Он должен выглядеть так:

{% include snippetcut_example path=".helm/templates/deployment.yaml" syntax="yaml" examples=page.examples %}

Здесь мы создали werf-шаблон для создания ресурса Deployment. Этот шаблон по сути является Helm-шаблоном, но с некоторой [дополнительной функциональностью]({{ site.url }}/documentation/v1.2/advanced/helm/overview.html), которую предлагает werf. Например, конструкция {% raw %}`image: {{ .Values.werf.image.app }}`{% endraw %} здесь используется для того, чтобы werf автоматически подставлял генерируемый тег образа (и имя образа) в поле `image`, так как тег образа становится известен только во время сборки.

Werf пересобирает образы только при изменениях в файлах, указанных в `werf.yaml`, а также при изменении самого `werf.yaml`. При пересборке изменится и тег образа, что приведёт к обновлению Deployment'а. Если же изменений в вышеупомянутых файлах нет, то образ не пересоберётся, а Deployment и создаваемые им ресурсы не передеплоятся, т.к. в этом просто нет необходимости: в кластере уже самая свежая версия приложения.

## Service

Ресурс Service позволяет другим приложениям в кластере обращаться к нашему приложению:

{% include snippetcut_example path=".helm/templates/service.yaml" syntax="yaml" examples=page.examples %}

## Ingress

Ресурс Ingress позволяет открыть доступ к нашему приложению *снаружи* кластера (в отличие от Service, который разрешает доступ только между приложениями *внутри* кластера). В Ingress'е мы указываем, на какой Service должен пойти внешний трафик, который попадает на домен `werf-guide-app`. Опишем наш Ingress-ресурс в файле `.helm/templates/ingress.yaml`:

{% include snippetcut_example path=".helm/templates/ingress.yaml" syntax="yaml" examples=page.examples %}

## Деплой в Kubernetes

> В случае, если вы удалили namespace приложения, то необходимо вернуться к предыдущей главе и заново пересоздать registry secret.

Команда [werf converge]({{ site.url }}/documentation/reference/cli/werf_converge.html) выполнит сразу и сборку, и развертывание приложения в Kubernetes:
```shell
werf converge --repo <имя пользователя Docker Hub>/werf-guide-app
```

Результат выполнения в случае успешной сборки и деплоя:
```shell
...
│ app/dockerfile  Successfully built 4c1054085159
│ │ app/dockerfile  Successfully tagged 93c05bf8-c459-4768-b388-3cdbc80e2868:latest
│ ├ Info
│ │       name: .../werf-guide-app:f4caaa836701e5346c4a0514bb977362ba5fe4ae114d0176f6a6c8cc-1612277803607
│ │       size: 371.4 MiB
│ └ Building stage app/dockerfile (40.31 seconds)
└ :boat: image app (41.13 seconds)
...
┌ Waiting for release resources to become ready
│ ┌ Status progress
│ │ DEPLOYMENT                                                                                                                                                      REPLICAS                      AVAILABLE                        UP-TO-DATE
│ │ app                                                                                                                                                        1/1                           1                                1
│ │ │   POD                                                           READY                  RESTARTS                       STATUS
│ │ └── 687f8cc569-n6gkw                                              1/1                    0                              Running
│ └ Status progress
└ Waiting for release resources to become ready (0.02 seconds)

NAME: werf-guide-app
LAST DEPLOYED: Tue Feb  2 21:57:23 2021
NAMESPACE: werf-guide-app
STATUS: deployed
REVISION: 1
TEST SUITE: None
Running time 62.66 seconds
```

Теперь обращаемся к нашему приложению:
```shell
curl http://werf-guide-app/ping
```

И получаем ответ:
```shell
pong
```
