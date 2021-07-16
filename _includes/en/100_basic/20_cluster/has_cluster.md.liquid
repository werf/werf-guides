{% comment %} TODO(lesikov): этот файл не трогал, надо переработать. Где-то здесь должен быть также указан чеклист, что должно быть в кластере развернуто (либо нормально описано, что ожидается от кластера). {% endcomment %}

## У меня уже есть кластер

Если у вас уже есть кластер, вероятно, вы уже сталкивались с его конфигурированием. Проверьте свой кластер по чек-листу в начале этой статьи.

Вне зависимости от того, как установлен кластер, необходимо получить ключи доступа к Kubernetes-кластеру в виде файла `.kube/config`.

<!--  .kube/config -->
{% offtopic title="Что за .kube/config?" %}
Это файл, хранящий реквизиты доступа к кластеру. Если разместить его по пути `~/.kube/config`, утилиты, работающие с Kubernetes, смогут подключиться к кластеру.

Файл выглядит примерно так:

{% snippetcut name=".kube/config" url="#" %}
{% raw %}
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: ALotOfNumbersAndLettersAndSoOnAveryVERYveryLongStringInBase64=
    server: https://127.0.0.1:6445
  name: kubernetes
contexts:
- context:
    cluster: kubernetes
    user: kubernetes-admin
  name: kubernetes-admin@kubernetes
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: kubernetes-admin
  user:
    client-certificate-data: ManyNumbersAndLettersAVERYveryVerylongString=
    client-key-data: ManyLettersAndNumbersAveryVeryVERYlongString=
```
{% endraw %}
{% endsnippetcut %}

Обратите внимание, что кроме ключей для подключения здесь содержится путь до API Kubernetes — например, `server: https://127.0.0.1:6445`. Важно, чтобы этот путь был корректным. Некоторые утилиты, генерирующие `.kube/config`, не подставляют туда публичный IP — тогда его нужно скорректировать вручную.

{% endofftopic %}
<!-- / .kube/config -->

### Проверка работоспособности и доступа к кластеру

Проверьте свой кластер по чек-листу в начале этой статьи.

Итогом всех манипуляций должна стать возможность получить доступ к кластеру с помощью утилиты `kubectl` (возможно, эту утилиту придётся установить отдельно). Например, вызов:

```bash
kubectl get ns
```

… покажет список всех namespace'ов в кластере, а не сообщение об ошибке.

### Ingress

В вашем кластере должен быть установлен Nginx Ingress. Если его нет, то в документации Kubernetes можно найти [подробные инструкции](https://kubernetes.github.io/ingress-nginx/deploy/).

### Registry

werf поддерживает [множество имплементаций]({{ site.url }}/documentation/advanced/supported_registry_implementations.html) registry.

Убедитесь, что есть доступ кластера по сети к registry.

### Hosts

В самоучителе предполагается, что наше приложение (вернее, его Nginx Ingress) будет доступно по адресу `werf-guide-app` и мы используем docker hub registry. В процессе прохождения статей самоучителя мы будем модифицировать файл hosts, в случае необходимости использования других доменных имен мы ожидаем, что вы самостоятельно выберете и настроите нужные DNS-записи, а при прохождении самоучителя будете подставлять в код ваши домены.

### Авторизация в Registry

Для того, чтобы werf смог загрузить собранный образ в registry, нужно на локальной машине авторизоваться с помощью `docker login` примерно так:

```bash
docker login <registry_domain> -u <account_login> -p <account_password>
```
