---
title: Деплой приложения
permalink: java_springboot/100_basic/30_deploy.html
examples: examples/basic/002_deploy
examples_initial: examples/basic/001_build
description: |
    В предыдущих главах мы собрали образ приложения и подготовили окружение для его развертывания. Теперь развернём приложение в ранее подготовленном кластере Kubernetes.

    При деплое в Kubernetes используются Kubernetes-манифесты, которые описывают ресурсы (объекты Kubernetes), необходимые для работы приложений. Эти ресурсы включают в себя, к примеру, Deployment, отвечающий за запуск приложений в контейнерах, и Service/Ingress, отвечающие за доступ к запущенным приложениям изнутри и извне кластера.
---

{% include 100_basic/30_deploy.md.liquid %}