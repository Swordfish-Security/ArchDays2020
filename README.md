# Материал для живой демонстрации на конференции ArchDays 2020

Ниже приведена инструкция по внедрению утилит проверки безопасности Docker-образов и Dockerfile на примере пайплайна в GitLab.
В качестве примера сборки уязвимого Docker-образа мы будем использовать исходники отсюда: https://github.com/Swordfish-Security/Pentest-In-Docker

**Обновляемся и ставим Docker:**

`sudo apt-get update && sudo apt-get install docker.io`

**Добавляем текущего пользователя в группу docker, чтобы можно было работать с ним без sudo и перелогиниваемся:**

`sudo addgroup %username% docker`

**Находим свой IP-адрес. Либо локальный:**

`ip addr`

**, либо внешний:**

`curl -s http://whatismyip.akamai.com`

**Запускаем инстанс GitLab, указывая наш IP-адрес:**

    docker run --detach \
    --hostname %IP-ADDRESS% \
    --publish 443:443 --publish 80:80 \
    --name gitlab \
    --restart always \
    --volume /srv/gitlab/config:/etc/gitlab \
    --volume /srv/gitlab/logs:/var/log/gitlab \
    --volume /srv/gitlab/data:/var/opt/gitlab \
    gitlab/gitlab-ce:latest

**За процессом установки можно следить через логи Docker:**

`docker logs -f gitlab`

**Открываем GitLab в браузере и задаём пароль для пользователя root:**

http://%IP-ADDRESS%/

**Импортируем файлы для нового проекта из нашего репозитория:**

http://%IP-ADDRESS%/projects/new
*Git Repository URL:* https://github.com/Swordfish-Security/ArchDays2020

**Переходим в консоль и скачиваем gitlab-runner, который будет запускать все необходимые операции по сканированию:**

`sudo curl -L --output /usr/local/bin/gitlab-runner https://gitlab-runner-downloads.s3.amazonaws.com/latest/binaries/gitlab-runner-linux-amd64`

**Делаем бинарный файл runner'a исполняемым:**

`sudo chmod +x /usr/local/bin/gitlab-runner`

**Добавляем пользователя ОС для gitlab-runner'а, устанавливаем и запускаем сервис:**

    sudo useradd --comment 'GitLab Runner' --create-home gitlab-runner --shell /bin/bash && \
    sudo gitlab-runner install --user=gitlab-runner --working-directory=/home/gitlab-runner && \
    sudo gitlab-runner start

**Убеждаемся, что сервис успешно запустился:**

`sudo gitlab-runner status`

**Регистрируем наш сервис в Gitlab. Для этого берем параметры URL и token со страницы Runner'ов:**

http://%IP-ADDRESS%/root/ArchDays2020/-/settings/ci_cd#js-runners-settings

*URL:*
http://%IP-ADDRESS%/
*Token:*
%TOKEN%

**Непосредственно регистрируем сервис, используя полученные значения:**

    sudo gitlab-runner register -n \
    --url "http://%IP-ADDRESS%/" \
    --registration-token "%TOKEN%" \
    --executor docker \
    --description "docker-in-docker" \
    --docker-image "docker:19.03.12" \
    --docker-privileged \
    --tag-list "docker,privileged"

**Включаем Runner'ы для текущего проекта, если они ещё не включены:**

http://%IP-ADDRESS%/root/ArchDays2020/-/settings/ci_cd#js-runners-settings

**Конфигурация самого GitLab завершена, и наш репозиторий содержит файл конфигурации пайплайна, который можно выполнить.**
Для целей нашего живого демо мы уже задали все параметры, которые потребуются, чтобы просканировать собранное уязвимое приложение и его Dockerfile.
Теперь достаточно запустить пайплайн на выполнение: 

http://%IP-ADDRESS%/root/ArchDays2020/-/pipelines/new

В результате выполнения все найденные проблемы будут доступны как в виде отдельных артефактов в каждой задаче (*.json), так и в виде простого html-отчёта, который можно найти в задаче Report

(Спасибо участнику George за комментарий) - стоит отдельно упомянуть, что в этом пайплайне не рассматривается более безопасный вариант установки утилит (в частности например сборка собственных небольших образов с конкретными версиями Hadolint/Dockle/Trivy), чтобы избежать риска злонамеренной подмены бинарных файлов при выкачивании их из внешнего репозитория и перевод GitLab на https. Также надо понимать, что в данном пайплайне используется привилегированный контейнер для GitLab и поэтому необходимо ограничить доступ к сети, где он развернут.

P.S. Дополнительную информацию по другим способам внедрения этих утилит и более детальные инструкции по установке можно почерпнуть из моей статьи на Хабре: https://habr.com/ru/company/swordfish_security/blog/524490/
