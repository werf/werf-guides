---
title: Локальная разработка
permalink: golang/300_dev_process/10_local.html
layout: development
---


TODO: Есть несколько способов (run, compose, converge). Какой выбрать зависит от:

- Наличия зависимостей
- Компиляции
- Времени сборки
- Размере образа
- ???




Табличка-выбор

                                    ран             компоуз         конвердж_в_локальный









В чем разница между werf run, werf compose, werf converge в локальный minikube и werf converge в удалённый кластер?

Что из этого лучше всего применимо для nodeJS


-----------



## Видеть результат

Если registry и стенд для разворачивания результата находится на локальном компьютере — мы экономим время на перекачку собранных образов по сети. В качестве площадки для запуска контейнеров можно использовать:

- minikube — более требователен к ресурсам, хотя и ближе к "настоящему" кластеру
- docker compose — требует меньше ресурсов и меньшей квалификации

Настройку minikube мы рассмотрели в главе ["Подготовка кластера"](../100_basic/20_cluster.html). Однако, werf позволяет напрямую работать с docker compose, дополняя его возможности.

Если вы переводите существующее приложение на werf и в рамках главы "Ускорение сборки" уже перешли целиком на `werf.yaml` - возможно вы захотите задействовать готовый `docker-compose.yml`.

Допустим, ваш `docker-compose.yml` запускает два образа. Один — ранее собиравшийся через `Dockerfile`, второй — с бд Redis:

{% snippetcut name="docker-compose.yml" url="#" %}
{% raw %}
```yaml
version: "3.8"
services:
  basicapp:
    build: .
    ports:
      - "5000:5000"
    depends_on:
      - redis
  redis:
    image: "redis:alpine"
```
{% endraw %}
{% endsnippetcut %}

Скорректируйте описание того, откуда берётся образ для `basicapp`, подставив туда переменную окружения, которую сгенерирует werf:

{% snippetcut name="docker-compose.yml" url="#" %}
{% raw %}
```yaml
version: "3.8"
services:
  basicapp:
    image: "$WERF_IMAGE_BASICAPP"
    ports:
      - "5000:5000"
    depends_on:
      - redis
  redis:
    image: "redis:alpine"
```
{% endraw %}
{% endsnippetcut %}

Для каждого `image` в `werf.yaml` будет доступно по одной переменной вида `$WERF_IMAGE_{НАЗВАНИЕ ОБРАЗА КАПСОМ}`.

Закоммитьте изменения в git и запустите compose:

```shell
werf compose up --follow
```

Образы запустятся и будут пересобираться и перезапускаться при каждом коммите.
