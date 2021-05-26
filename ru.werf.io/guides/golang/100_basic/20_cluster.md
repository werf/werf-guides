---
title: Подготовка окружения
permalink: golang/100_basic/20_cluster.html
---

Чтобы вести разработку в среде Kubernetes, необходимо:

- локально установленный werf — сделано на предыдущих шагах;
- приложение в Git — сделано на предыдущих шагах;
- кластер Kubernetes;
- реестр контейнеров (container registry);
- правильные DNS-записи.

**Настройка окружения — это сложная задача**. Не рекомендуем закапываться в решение проблем своими руками: используйте существующие на рынке услуги, позовите на помощь коллег, которые имеют практику настройки инфраструктуры, или задавайте вопросы в [Telegram-сообществе werf](https://t.me/werf_ru).

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

<div class="tabs">
<a href="javascript:void(0)" class="tabs__btn tabs__install__btn" onclick="openTab(event, 'tabs__install__btn', 'tabs__install__content', 'tab__install__minikube')">Minikube</a>
<a href="javascript:void(0)" class="tabs__btn tabs__install__btn" onclick="openTab(event, 'tabs__install__btn', 'tabs__install__content', 'tab__install__docker')">Docker Desktop</a>
<a href="javascript:void(0)" class="tabs__btn tabs__install__btn" onclick="openTab(event, 'tabs__install__btn', 'tabs__install__content', 'tab__install__cloud')">Использование облачного провайдера</a>
<a href="javascript:void(0)" class="tabs__btn tabs__install__btn" onclick="openTab(event, 'tabs__install__btn', 'tabs__install__content', 'tab__install__ihave')">У меня уже есть кластер</a>
</div>

<div id="tab__install__minikube" class="tabs__content tabs__install__content" markdown="1">
{% include_relative 20_cluster_minikube.md %}
</div>

<div id="tab__install__docker" class="tabs__content tabs__install__content" markdown="1">
{% include_relative 20_cluster_docker_desktop.md %}
</div>

<div id="tab__install__cloud" class="tabs__content tabs__install__content" markdown="1">
{% include_relative 20_cluster_cloud_provider.md %}
</div>

<div id="tab__install__ihave" class="tabs__content tabs__install__content" markdown="1">
{% include_relative 20_cluster_has_cluster.md %}
</div>

## Финальные проверки

Если вы уже убедились в работоспособности самого кластера, пришло время проверить registry и ingress. Эти команды:

```shell
docker pull ubuntu:18.04
docker tag ubuntu:18.04 registry.example.com:5000/ubuntu:18.04
docker push registry.example.com:5000/ubuntu:18.04
```

… должны успешно загрузить образ Ubuntu в registry и не выдать ошибку. А эта команда:

```shell
curl example.com
```

… должна выдать страницу ошибки nginx ingress (если вы ещё не задеплоили приложения в кластер).

<div id="go-forth-button">
    <go-forth url="30_deploy.html" label="Деплой приложения" framework="{{ page.label_framework }}" ci="{{ page.label_ci }}" guide-code="{{ page.guide_code }}" base-url="{{ site.baseurl }}"></go-forth>
</div>
