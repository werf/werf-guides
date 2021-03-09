---
title: Подготовка окружения
permalink: nodejs/100_basic/20_cluster.html
---

Чтобы вести разработку в среде Kubernetes, необходимо:

- локально установленный werf — сделано на предыдущих шагах;
- приложение в Git — сделано на предыдущих шагах;
- кластер Kubernetes;
- реестр контейнеров (container registry);
- правильные DNS-записи.

**Настройка окружения — это сложная задача**. Не рекомендуем закапываться в решение проблем своими руками: используйте существующие на рынке услуги, позовите на помощь коллег, которые имеют практику настройки инфраструктуры, или задавайте вопросы в [Telegram-сообществе werf](https://t.me/werf_ru).

{% offtopic title="Чек-лист для самопроверки окружения" %}

- У вас есть кластер Kubernetes (версии 1.14 или выше).
    - В кластере установлен ingress-контроллер и на него направлен домен `example.com`, т.е. туда можно заходить браузером.
- У вас есть registry.
    - Registry доступен по домену `registry.example.com`.
    - Кластер может pull'ить образы из вашего registry.
- С локального компьютера:
    - есть доступ в кластер (работает `kubectl version`);
    - есть доступ в registry (работает `docker push`);
    - в браузере открывается `example.com` (пусть и показывает 404 от Ingress).

{% endofftopic %}

Выберите, как будете реализовывать окружение:

<div style="display: flex; justify-content: space-between; margin: 0 10px 0 20px;">
<div class="button__blue button__blue_inline expand_columns_button" id="minikube_button"><a href="#">Minikube</a></div>
<div class="button__blue button__blue_inline expand_columns_button" id="docker_desktop_button"><a href="#">Docker Desktop</a></div>
<div class="button__blue button__blue_inline expand_columns_button" id="kind_button"><a href="#">Kind</a></div>
<div class="button__blue button__blue_inline expand_columns_button" id="cloud_provider_button"><a href="#">Использование облачного провайдера</a></div>
<div class="button__blue button__blue_inline expand_columns_button" id="has_cluster_button"><a href="#">У меня уже есть кластер</a></div>
</div>

{% expandonclick id="minikube_button__content" %}
{% include_relative 20_cluster_minikube.md %}
{% endexpandonclick %}

{% expandonclick id="docker_desktop_button__content" %}
{% include_relative 20_cluster_docker_desktop.md %}
{% endexpandonclick %}

{% expandonclick id="kind_button__content" %}
{% include_relative 20_cluster_kind.md %}
{% endexpandonclick %}

{% expandonclick id="cloud_provider_button__content" %}
{% include_relative 20_cluster_cloud_provider.md %}
{% endexpandonclick %}

{% expandonclick id="has_cluster_button__content" %}
{% include_relative 20_cluster_has_cluster.md %}
{% endexpandonclick %}

## Финальные проверки

Если вы уже убедились в работоспособности самого кластера, пришло время проверить registry и Ingress. Эти команды:

```shell
docker tag ubuntu:18.04 registry.example.com/ubuntu:18.04
docker push registry.example.com/ubuntu:18.04
```

… должны успешно загрузить образ Ubuntu в registry и не выдать ошибку. А эта команда:

```shell
curl example.com
```

… должна выдать страницу ошибки nginx Ingress (если вы ещё не задеплоили приложения в кластер).

<div id="go-forth-button">
    <go-forth url="30_deploy.html" label="Деплой приложения" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
