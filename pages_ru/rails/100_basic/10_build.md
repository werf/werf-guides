---
title: Сборка образа
permalink: rails/100_basic/10_build.html
---

{% include 100_basic/10_build.md.liquid %}

В этой главе мы соберём Docker-образ с демо-приложением, используя werf и [Dockerfile](https://docs.docker.com/engine/reference/builder/), а потом проверим собранный образ, запустив его локально.

## Подготовка

Установите werf и его зависимости, [следуя инструкциям]({{ site.url }}/installation.html).

{% offtopic title="В Windows пользователю также понадобятся права на создание символьных ссылок" %}
1. В PowerShell выполните для текущего пользователя с правами администратора:
```powershell
$ntprincipal = new-object System.Security.Principal.NTAccount "$env:UserName"
$sidstr = $ntprincipal.Translate([System.Security.Principal.SecurityIdentifier]).Value.ToString()
$tmp = [System.IO.Path]::GetTempFileName()
secedit.exe /export /cfg "$($tmp)"
$currentSetting = ""
foreach($s in (Get-Content -Path $tmp)) {
    if ($s -like "SECreateSymbolicLinkPrivilege*") {
        $x = $s.split("=",[System.StringSplitOptions]::RemoveEmptyEntries)
        $currentSetting = $x[1].Trim()
    }
}
if ($currentSetting -notlike "*$($sidstr)*") {
    if ([string]::IsNullOrEmpty($currentSetting)) {
        $currentSetting = "*$($sidstr)"
    } else {
        $currentSetting = "*$($sidstr),$($currentSetting)"
    }
    $tmp2 = [System.IO.Path]::GetTempFileName()
    @"
[Unicode]
Unicode=yes
[Version]
signature="`$CHICAGO`$"
Revision=1
[Privilege Rights]
SECreateSymbolicLinkPrivilege = $($currentSetting)
"@ | Set-Content -Path $tmp2 -Encoding Unicode -Force
    cd (Split-Path $tmp2)
    secedit.exe /configure /db "secedit.sdb" /cfg "$($tmp2)" /areas USER_RIGHTS
}
```
1. Для применения изменений перезайдем в учетную запись Windows, либо выполним команду:
```powershell
gpupdate /force
```
{% endofftopic %}

## Создадим новый репозиторий с демо-приложением

Все дальнейшие команды потребуется выполнять в PowerShell (для Windows) или Bash (для macOS и Linux).

В отдельной директории на своём компьютере выполним команды:
```bash
git clone https://github.com/werf/werf-guides
cp -r werf-guides/examples/rails/000_app rails-app
cd rails-app
git init
git add .
git commit -m "initial"
```

## Создадим Dockerfile

> В Windows во избежание проблем при редактировании файлов рекомендуем использовать [Notepad++](https://notepad-plus-plus.org/downloads/) или любой другой редактор вместо стандартного Блокнота.

Реализуем логику сборки нашего приложения с [Dockerfile](https://docs.docker.com/engine/reference/builder/). Для этого добавим следующий `Dockerfile` в нашу рабочую директорию:

```shell
cp ../werf-guides/examples/rails/010_build/Dockerfile .
```

{% snippetcut name="Dockerfile" url="https://github.com/werf/werf-guides/blob/master/examples/rails/010_build/Dockerfile" %}
{% include_file "examples/rails/010_build/Dockerfile" syntax="Dockerfile" %}
{% endsnippetcut %}

## Интеграция werf и Dockerfile

Создадим в корне репозитория основной файл конфигурации werf `werf.yaml`, в котором укажем, какой `Dockerfile` должен будет использоваться при сборке с werf:

```shell
cp ../werf-guides/examples/rails/010_build/werf.yaml .
```

{% snippetcut name="werf.yaml" url="https://github.com/werf/werf-guides/blob/master/examples/rails/010_build/werf.yaml" %}
{% include_file "examples/rails/010_build/werf.yaml" %}
{% endsnippetcut %}

В `werf.yaml` может описываться сборка сразу нескольких образов. Также для сборки образа существует ряд дополнительных настроек, с которыми можно ознакомиться в [документации]({{ site.url }}/documentation/reference/werf_yaml.html#dockerfile-image-section-image).

## Сборка с werf

Перед выполнением сборки необходимо добавить наши изменения в коммит:
```bash
git add werf.yaml Dockerfile
git commit -m "Add build configuration"
```

> Чуть позже мы разберём, для чего изменения нужно добавлять в коммит перед сборкой/деплоем и как обойтись без постоянного создания новых коммитов при локальной разработке.

Запустим сборку командой [`werf build`]({{ site.url }}/documentation/reference/cli/werf_build.html):
```bash
werf build
```

Результат выполнения команды при успешной сборке:
{% raw %}
```bash
┌ ⛵ image basicapp
│ ┌ Building stage basicapp/dockerfile
│ │ basicapp/dockerfile  Sending build context to Docker daemon  11.64MB
│ │ basicapp/dockerfile  Step 1/14 : FROM ruby:2.7.1
│ │ basicapp/dockerfile   ---> d8ca85855516
│ │ basicapp/dockerfile  Step 2/14 : WORKDIR /app
│ │ basicapp/dockerfile   ---> Using cache
...
│ │ basicapp/dockerfile  Successfully built 3a4ede4e9556
│ │ basicapp/dockerfile  Successfully tagged 0041b344-efe4-416d-baff-5e50fbb712b0:latest
│ ├ Info
│ │       name: werf-guided-rails:31e0e7436c3055fa816fc770ebda185bacb7e8ef53775b8e5488a83f-1611855308907
│ │       size: 929.2 MiB
│ └ Building stage basicapp/dockerfile (94.47 seconds)
└ ⛵ image basicapp (96.07 seconds)

Running time 96.38 seconds
```
{% endraw %}

## Запуск приложения

Запустить контейнер локально на основе собранного образа можно командой [werf run]({{ site.url }}/documentation/cli/main/run.html):
```bash
werf run --docker-options="--rm -p 3000:3000" basicapp -- bash -ec "bundle exec rails db:migrate && bundle exec puma"
```

Здесь [параметры Docker](https://docs.docker.com/engine/reference/run/) мы задали опцией `--docker-options`, а команду для выполнения в контейнере указали в конце, после двух дефисов.

Теперь приложение доступно на [http://127.0.0.1:3000/](http://127.0.0.1:3000/).
