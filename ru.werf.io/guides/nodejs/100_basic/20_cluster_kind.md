## Kind

Это инсталлятор локальных кластеров Kubernetes, узлы в которых запускаются в виде Docker-контейнеров.

### Установка

Скачайте kind [последней версии](https://github.com/kubernetes-sigs/kind/releases/latest). kind умеет запускать кластер из нескольких узлов, об этом можно прочесть в [документации](https://kind.sigs.k8s.io/).

Для целей самоучителя достаточно:
- запустить локальный Registry
- создать кластер из одного узла
- установить в кластере Ingress-контроллер на основе nginx.

### Registry

У kind нет команды для запуска registry. Самый простой способ — запустить его вручную из Docker-образа:

```shell
docker run -d -p 5000:5000 --restart always --name registry registry:2
```

Обратите внимание, что в этом случае registry будет работать без SSL-шифрования. А значит, при работе с werf потребуется добавлять [параметр](https://werf.io/documentation/reference/cli/werf_managed_images_add.html#options) `--insecure-registry=true`.

### Запуск кластера

kind создаёт кластер на основе конфигурационного файла, в котором можно изменить настройки создаваемого кластера. Чтобы кластер увидел наш запущенный registry, нужно изменить настройки containerd c помощью ключа `containerdConfigPatches`. Для Ingress нужен лейбл на узле, где будет запускаться nginx, и маппинг для портов 80 и 443 — это делается ключами `kubeadmConfigPatches` и `extraPortMappings`.

Теперь можно создать кластер:

```
cat <<EOF | kind create cluster --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."registry.example.com:5000"]
      endpoint = ["http://registry:5000"]
nodes:
  - role: control-plane
    kubeadmConfigPatches:
    - |
      kind: InitConfiguration
      nodeRegistration:
        kubeletExtraArgs:
          node-labels: "ingress-ready=true"
    extraPortMappings:
    - containerPort: 80
      hostPort: 80
      protocol: TCP
    - containerPort: 443
      hostPort: 443
      protocol: TCP
EOF
```

В результате автоматически будет создан файл `.kube/config` с ключами доступа к локальному кластеру.

Итогом этих манипуляций должна стать возможность получить доступ к кластеру с помощью утилиты `kubectl` (возможно, эту утилиту придётся установить отдельно). Например, вызов:

```shell
kubectl get ns
```

… покажет список всех namespace в кластере, а не сообщение об ошибке.

# Доступ к registry из кластера

В конфигурации кластера мы прописали такой параметр

```
endpoint = ["http://registry:5000"]
```

Здесь `registry` — это имя Docker-контейнера, в котором запущен registry. Чтобы кластер «увидел» этот контейнер, нужно подключить контейнер к Docker-сети кластера. Для этого нужно выполнить команду:

```
docker network connect "kind" registry
```

Здесь `kind` — имя Docker-сети кластера, а "registry" — имя контейнера с registry.

# Ingress

У kind нет команды для установки Ingress, поэтому [установим Nginx Ingress вручную](https://kubernetes.github.io/ingress-nginx/deploy/).

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/static/provider/kind/deploy.yaml
```

Дождаться установки можно такой командой:

```
  kubectl wait --namespace ingress-nginx \
    --for=condition=ready pod \
    --selector=app.kubernetes.io/component=controller \
    --timeout=90s
```

{% offtopic title="Как убедиться, что с Ingress всё хорошо?" %}

По умолчанию можно считать, что всё хорошо. Если не уверены, что тут написано, вернитесь к этому пункту позже.

- Балансировщик установлен.
- Pod с балансировщиком корректно поднялся (для nginx-ingress это можно посмотреть так: `kubectl -n ingress-nginx get po`).
- На 80-м порту (это можно посмотреть с помощью `lsof -n | grep LISTEN`) работает нужное приложение.

{% endofftopic %}


### Hosts

В самоучителе предполагается, что кластер (вернее, его Nginx Ingress) доступен по адресу `example.com`, а registry — по адресу `registry.example.com`. Именно этот домен и его поддомены указаны в дальнейшем в конфигах. В случае, если вы будете использовать другие адреса, скорректируйте конфигурацию самостоятельно.

Пропишите в локальном файле `/etc/hosts` строки вида:

```
127.0.0.1           example.com
127.0.0.1           registry.example.com
```

### Авторизация в Registry

Если вы запустили registry локально, как приведено в примере выше, то ваш локальный registry работает без пароля и ничего делать не нужно.

{% offtopic title="Я использую внешний registry или установил логин/пароль" %}
Для того, чтобы werf смог загрузить собранный образ в registry, нужно на локальной машине авторизоваться с помощью `docker login` примерно так:

```shell
docker login <registry_domain> -u <account_login> -p <account_password>
```
{% endofftopic %}
