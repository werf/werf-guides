## Minikube

Это «облегчённая» версия Kubernetes, работающая только на одном узле.

### Установка

[Установите minikube](https://minikube.sigs.k8s.io/docs/start/) и запустите его:

```shell
minikube start --driver=docker
```

**ВАЖНО:** Если Minikube уже запущен в вашей системе, то надо удостоверится, что используется driver под названием `docker`. Если нет, то требуется перезапустить Minikube с помощью команды `minikube delete` и команды для старта, показанной выше.

В результате автоматически будет создан файл `.kube/config` с ключами доступа к локальному кластеру и werf сможет подключиться к локальному registry.

Итогом этих манипуляций должна стать возможность получить доступ к кластеру с помощью утилиты `kubectl` (возможно, эту утилиту придётся установить отдельно). Например, вызов:

```shell
kubectl get ns
```

… покажет список всех namespace'ов в кластере, а не сообщение об ошибке.

### Ingress

В Minikube нужно [включить addon](https://kubernetes.io/docs/tasks/access-application-cluster/ingress-minikube/#enable-the-ingress-controller):
```shell
minikube addons enable ingress
```

{% offtopic title="Как убедиться, что с Ingress всё хорошо?" %}

По умолчанию можно считать, что всё хорошо. Если не уверены, что тут написано, вернитесь к этому пункту позже.

- Балансировщик установлен.
- Pod с балансировщиком корректно поднялся (для nginx-ingress это можно посмотреть так: `kubectl -n ingress-nginx get po`).
- На 80-м порту (это можно посмотреть с помощью `lsof -n | grep LISTEN`) работает нужное приложение.

{% endofftopic %}

### Registry

Включите addon minikube registry:

```shell
minikube addons enable registry
```

И далее, в зависимости от ОС:

{% offtopic title="Windows" %}
{% include_relative 20_cluster_minikube_registry_win.md %}
{% endofftopic %}
{% offtopic title="MacOS" %}
{% include_relative 20_cluster_minikube_registry_macos.md %}
{% endofftopic %}
{% offtopic title="Linux" %}
{% include_relative 20_cluster_minikube_registry_linux.md %}
{% endofftopic %}

### Hosts

В самоучителе предполагается, что кластер (вернее, его Nginx Ingress) доступен по адресу `example.com`, а registry — по адресу `registry.example.com`. Именно этот домен и его поддомены указаны в дальнейшем в конфигах. В случае, если вы будете использовать другие адреса, скорректируйте конфигурацию самостоятельно.

Обновите локальный файл `/etc/hosts`:

```shell
printf "\n$(minikube ip) example.com\n127.0.0.1 registry.example.com\n" | sudo tee -a /etc/hosts
```

### Авторизация в Registry

Registry, установленный в minikube, не требует авторизации. Никаких дополнительных действий предпринимать не нужно.
