---
title: Периодическое выполнение заданий
permalink: rails/200_real_apps/60_cron.html
layout: wip
---

В этой главе мы покажем как запускать периодические фоновые задания (job) на примере rails rake tasks, а также расскажем что такое CronJob, чем он отличается от Job и для чего ещё можно использовать CronJob, а также ответим на часто возникающие вопросы связанные с CronJob.

<cut>

<!-- TODO: Надо сделать шаг подготовка сворачиваемым и по умолчанию свёрнутым -->

## Подготовка

Возьмём за основу наше приложение. Состояние директории `rails-app` должно соответствовать шагу `examples/rails/019_fixup_consistency`:

```shell
git clone https://github.com/werf/werf-guides
cp -r werf-guides/examples/rails/019_fixup_consistency rails-app
cd rails-app
git init
git add .
git commit -m "initial"
```
</cut>

Необходимо добавить dns-запись в файл hosts:

```shell
echo "$(minikube ip) debug-mails.example.com" | sudo tee -a /etc/hosts
```

## Запускаем фоновые job

Рассмотрим подробнее задания, которые будут реализованы для регулярного выполнения в фоне.

### Подготовка приложения
Чтобы добавить job в наше приложение нам необходимо скопировать файлы с кодом в наш репозиторий:

```shell
cp ../werf-guides/examples/rails/800_cron/app/controllers/api/labels_controller.rb app/controllers/api/labels_controller.rb
cp ../werf-guides/examples/rails/800_cron/app/mailers/notifications_mailer.rb app/mailers/notifications_mailer.rb
cp ../werf-guides/examples/rails/800_cron/app/views/notifications_mailer/labels_count_report_email.html.erb app/views/notifications_mailer/labels_count_report_email.html.erb
cp ../werf-guides/examples/rails/800_cron/config/application.rb config/application.rb
cp ../werf-guides/examples/rails/800_cron/config/routes.rb config/routes.rb
cp ../werf-guides/examples/rails/800_cron/.helm/templates/cleanup-labels.yaml .helm/templates/cleanup-labels.yaml
cp ../werf-guides/examples/rails/800_cron/.helm/templates/debug-mails.yaml .helm/templates/debug-mails.yaml
cp ../werf-guides/examples/rails/800_cron/.helm/templates/ingress.yaml .helm/templates/ingress.yaml
cp ../werf-guides/examples/rails/800_cron/.helm/templates/send-report.yaml .helm/templates/send-report.yaml
cp ../werf-guides/examples/rails/800_cron/lib/tasks/crons.rake lib/tasks/crons.rake
```

### Задание по очистке устаревших записей в таблице labels

Job очищает записи в таблице labels с истекшим time to live имеющих срок жизни в 3 минуты.
Задача может работать полностью автономно и не требует доступа к запущенному http-серверу.
Особенность задачи в том, что она может долго выполняться в фоне, поэтому её выполнение внутри запущенного http-сервера не желательно.
За реализацию задачи по очистке отвечает rake task `crons:cleanup_labels`.
Применение job производится из директории проекта командой `rake crons:cleanup_labels`.

Регулярный запуск job реализован через CronJob (о том, что это такое напишем далее) с именем `cleanup-labels` с периодичностью в одну минуту.
CronJob будет создавать Job по описанному jobTemplate. Затем при помощи этих Job будут создавать Pod-ы, затем каждый запускаемый Pod выполнит в отдельном контейнере команду `rake crons:cleanup_labels`.

Необходимые для реализации job helm-файл и rake-файлы:

[Rake-файл](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/lib/tasks/crons.rake), содержащий инструкции для cron

[Helm-файл](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/.helm/templates/cleanup-labels.yaml), необходимый для деплоя этого job в наше приложение.

### Задание по периодической отсылке отчётов по почте

Job отправляет e-mail с текущим количеством labels администратору системы, реализовано внутри http-сервера и инициируется специальным запросом по api: `/api/send-report`, суть job состоит в том чтобы сделать post-запрос на бекенд basicapp.
Во избежание избыточного выполнения job одновременно на нескольких репликах инициировать его необходимо вне http-сервера с нашим приложением.

{% offtopic title="А что если я хочу запускать CronJob в том же поде и контейнере, где работает наш http-сервер?" %}
Так делать не принято и через CronJob в чистом виде не получится.
Таким способом реализована очистка устаревших записей в таблице labels [описанная выше](#задание-по-очистке-устаревших-записей-в-таблице-labels).
Для реализации такой схемы фоновое задание встраивается в само приложение (http-сервер в нашем случае).
Инициировать выполнение этого задания можно либо средствами самого приложения, либо через CronJob (предпочтительный вариант, описанный ниже).
{% endofftopic %}

Внутри приложения за реализацию отправки отчетов отвечает метод контроллера labels `send_report`. Отправка происходит из самого http-сервера и реализована через специальный дебаг-smtp-сервер (mailhog), который деплоится вместе с нашим приложением по адресу `debug-mails.example.com` и позволяет просматривать перехваченные письма.
Регулярный запуск job реализован через CronJob с именем `send-report` который будет создавать job по описанному jobTemplate и периодически запускаться каждую минуту.
Каждая из job будет создавать pod который будет выполнять в отдельном контейнере запрос на http-сервер `/api/send-report`, в ответ на который сервер выполнит отправку -mail на почту администратору.
  
Необходимые для реализации job helm-файлы и исходный код:
[labels_controller.rb](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/app/controllers/api/labels_controller.rb);
[notifications_mailer.rb](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/app/mailers/notifications_mailer.rb);
[labels_count_report_email.html.erb](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/app/views/notifications_mailer/labels_count_report_email.html.erb);
[debug-mails.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/.helm/templates/debug-mails.yaml);
[ingress.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/.helm/templates/ingress.yaml);
[send-report.yaml](https://github.com/werf/werf-guides/tree/master/examples/rails/800_cron/.helm/templates/send-report.yaml).

### Деплой и проверка работоспособности job

Закоммитим изменения:

```shell
git add .
git commit -m "Add cronjobs"
```

Запустим выкат:

```shell
werf converge --repo <имя пользователя Docker Hub>/werf-guided-rails
```

#### Проверим как выполняется очистка устаревших labels

Наш job удаляет labels, время жизни которых больше 3х минут.
Перед выполнением задания создадим тестовые данные:

```shell
curl -XPOST "http://example.com/api/labels?label=aaa"
curl -XPOST "http://example.com/api/labels?label=bbb"
curl -XPOST "http://example.com/api/labels?label=ccc"
curl -XPOST "http://example.com/api/labels?label=ddd"
curl -XPOST "http://example.com/api/labels?label=eee"
curl -XPOST "http://example.com/api/labels?label=fff"
curl -XPOST "http://example.com/api/labels?label=ggg"
```

Проверим список вновь созданных labels:
    
```shell
curl "http://example.com/api/labels"
```

Для проверки работы Cronjob, после его деплоя необходимо периодически запускать следующую команду до тех пор пока не увидим новые поды с префиксом в имени `cleanup-labels-`:

```shell
kubectl -n werf-guided-rails get cj,job,pod
```

Просмотрим логи одного из вновь созданных подов:

```shell
kubectl -n werf-guided-rails logs -f pod/cleanup-labels-<podId>
```

Спустя 3 минуты снова проверим список существующих labels, который должен быть пустым:

``shell
curl "http://example.com/api/labels"
```

#### Проверим как выполняется отправка отчетов

Проверять отправленные e-mail мы будем через smtp-сервер, который задеплоен с нашим приложением на предыдущем шаге. Для тестирования наше приложение настроено на отправку писем через этот smtp-сервер.

Перейдём по адресу [http://debug-mails.example.com](http://debug-mails.example.com).
Каждую минуту должно приходить новое письмо с отчетом о текущем количестве записей в таблице Labels.

## Что такое CronJob

**Job**
Каждый экземпляр job представляет собой одно фоновое выполнение какой-то задачи.
Запускаемая задача будет работать в отдельном pod'е согласно описанию шаблона `spec.template`.
После того как job завершилась, она переходит в final состояние, при этом однажды созданная в кластере, она не может быть обновлёна или изменёна после создания.
Всё это означает что ресурс job — одноразовый, и чтобы инициировать повторное выполнение задачи необходимо создать новый экземпляр job но уже с другим именем.

**CronJob**
Нужен чтобы организовать периодическое выполнение одной фоновой задачи.
В нашем примере мы как раз используем CronJob. CronJob ­— такая же фоновая задача, но, в отличие от job - многоразовая. CronJob представляет собой надстройку над job, которая задаёт шаблон для генерации одноразовых job. Инструкции в секции `spec.schedule` позволяют задать периодичность в привычном формате crontab.

Если остались вопросы можете также ознакомиться с материалами про CronJob на официальном сайте kubernetes по ссылкам ниже.
[https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/](https://kubernetes.io/docs/tasks/job/automated-tasks-with-cron-jobs/);
[https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/).

## Для чего ещё может быть использован CronJob

Вот некоторые из вариантов использования:

- Периодическая индексация БД.
- Обработчик очереди заданий.
    - Задания поступают в некоторую очередь.
    - CronJob инициирует работу обработчика следующего задания из этой очереди.
- Инвалидация истекших пользовательских доступов в БД.
- Очистка временных данных и устаревших пользовательских сессий в БД.
- И другие.