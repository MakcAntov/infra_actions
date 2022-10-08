# infra_actions
*Учебный проект для изучения работы GitHub Actions (Яндекс Практикум)*

## Workflow через GitHub

**Workflow** — это набор команд, которые выполнятся в виртуальном окружении после того, как произойдёт какое-то событие-триггер

* Зайдите в свой репозиторий на GitHub и перейдите во вкладку Actions:
* Нажмите ссылку *"set up a workflow yourself"*: будет создана директория `.github/workflows/main.yml`.

```
name: CI
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run a one-line script
        run: echo Hello, world!
      - name: Run a multi-line script
        run: |
          echo Add other actions to build,
          echo test, and deploy your project.
```


**Любой workflow должен содержать, как минимум три ключа:**
* `name` - имя. На верхнем уровне в автоматически созданном шаблоне указан ключ `name` со значением `CI` — это название workflow
* `on` - описывает триггер — событие, которое должно произойти, чтобы workflow начал выполняться. В шаблоне указан ивент:
    * `push` - это событие *git push*, а `branches` — это ветка, в которую должен быть сделан *push*
    * `pull_requests` - триггер на *pull_request* сработает, когда кто-то сделает запрос на изменение кода в ветке *master*. 
  Можно, например, проверить, что предполагаемые изменения не содержат синтаксических ошибок, код написан по *PEP8* и проходит все тесты, а деплоить код по триггеру *pull_request* не обязательно. 
  Ознакомится с другими триггерами можно [тут](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows)
* `jobs` - объединяет поименованные блоки команд — задачи, которые будут выполняться при запуске *workflow*. 
Каждая задача-job описывается набором параметров. 
Каждый job запускается в изолированном окружении, которое создаёт сервис GitHub Actions. 
    * Настройки окружения определяются в ключе `runs-on`. 
  Это аналогично инструкции *FROM* в докерфайле.
    * Под ключом `steps` записывается перечень шагов, команд, которые будут выполнены в этом блоке. 
  Начало каждого шага обозначается символом `-`.
    * Каждому шагу можно дать имя — для этого применяется ключ `name`. Это необязательно, но желательно. 
    * В ключе `run` хранится команда, которая будет выполнена в терминале окружения.
    * Вместо `run` можно применять `uses` — для вызова actions. 
  *Actions* — это скрипты, которые можно написать заранее и вызывать по имени из разных *workflow*.
  Больше об этом написано [тут](https://docs.github.com/en/actions/creating-actions/about-custom-actions#types-of-actions)

**Добавление директории workflow в Git:**
* Сделайте коммит, чтобы директория `.github/workflows` попала в Git
* Сделайте слияние при необходимости - `git pull`

**Запуск workflow**

Чтобы запустить workflow из примера, надо сделать `push` в master (или main), тогда сработает инструкция `on` — и `jobs` начнут выполняться.
```
(venv) ... Dev/infra_action$ git add .
(venv) ... Dev/infra_action$ git commit -m 'ваш_комментарий'
(venv) ... Dev/infra_action$ git push
```

## Настоящий workflow: PEP8

**Начнём с чистого листа.**

В коде *workflow* для клонирования репозитория применён предустановленный *action [checkout](https://github.com/actions/checkout)*, а для установки Python-окружения — *action [setup-python](https://github.com/actions/setup-python)*. 
Команды для выполнения этих операций можно прописать вручную прямо в workflow, но проще применить готовые скрипты.

Для последовательного запуска команд применяется такой синтаксис:
```
run: |
  команда1
  команда2
  команда3
```
В шаге `Install dependencies` (англ. «установка зависимостей») сначала обновляется *pip*, затем устанавливаются необходимые пакеты для тестов. 
После этого идёт установка всех зависимостей из *requirements.txt*.

На шаге `Test with flake8 and django tests `запускается проверка *flake8*

Чтобы ограничить тестирование только заданными директориями, нужно - 
создать файл конфигурации `setup.cfg` в корневой директории проекта:
в нём можно задавать поведение различных команд и плагинов проекта.

В данном случае `setup.cfg` наполнен инструкциями:
```
[flake8]
ignore =
    W503,
    F811
exclude =
    tests/,
    */migrations/,
    venv/,
    env/
per-file-ignores =
    */settings.py:E501
max-complexity = 10
```
* `ignore` — позволяет игнорировать определённые правила при проверке.
* `exclude` — исключает из проверки перечисленные директории. 
* `per-file-ignores` — позволяет игнорировать определённые предупреждения и ошибки для перечисленных файлов. В приведённом примере для файла *settings.py* будет проигнорирована ошибка E501 (ограничение длины строки в 79 символов). 
* `max-complexity` — максимальное значение цикломатической сложности функции.

Осталось проверить: `flake8 .`

## Настоящий workflow: тесты

Любое приложение должно быть покрыто тестами. 
В целях обучения - уже созданы тесты и добавлены в директорию.
Подготовим всё остальное:
Чтобы тесты запустились, нужно дописать инструкцию в `main.yml`
```yaml
.
.
.
        # запуск проверки проекта по flake8
        python -m flake8
        # перейти в папку, содержащую manage.py — 
        #<корневая_папка_infra_actions>/<папка_проекта>/manage.py
        cd infra_project/
        # запустить написанные разработчиком тесты
        python manage.py test
```

## Настоящий workflow: сборка docker-образа

На сервере проект должен запускаться в docker-контейнере. 
Чтобы каждый раз после изменений не переносить Dockerfile на сервер вручную или отдельным скриптом, удобно отправлять изменённый образ в репозиторий Docker Hub, используя инструкции в workflow.

Dockerfile для проекта уже подготовлен. 
Для запуска контейнера вам нужен image id образа: 
получить его можно командой `docker image ls`.

Соберите образ и запустите контейнер; команды должны выполняться из директории, в которой лежит Dockerfile:
```
docker build . # Соберёт образ на основе Dockerfile
docker image ls # Отобразит информацию обо всех образах
docker run -p 5000:5000 <IMAGE ID> # Запустит контейнер из образа с <IMAGE ID>
```
**Автоматическая сборка и пуш на Docker Hub**

По умолчанию все *jobs* в *workflow* запускаются одновременно. 
Однако обычно требуется последовательный запуск задач: 
например, *job* `build` надо запускать только после успешного выполнения задачи `tests`, а `deploy` — только по окончании `build`. 

Чтобы определить последовательность запуска задач, в workflow применяют ключ `needs`: 
в нём указывается имя того *job*, после выполнения которого должна запуститься описываемая задача.

Добавим в *workflow* новую задачу, в ней будет описана автоматическая пересборка и пуш обновлённого образа на Docker Hub. 
Эта задача должна выполняться только после успешного прохождения тестов.
```yaml
# .github/workflows/main.yml

# Тут ваши задачи тестирования и сборки образа
# ...
# Сразу после них добавьте новую задачу: деплой приложения
build_and_push_to_docker_hub:
    name: Push Docker image to Docker Hub
    runs-on: ubuntu-latest
    needs: tests
    steps:
      - name: Check out the repo
        # Проверка доступности репозитория Docker Hub для workflow
        uses: actions/checkout@v2 
      - name: Set up Docker Buildx
        # Вызов сборщика контейнеров docker
        uses: docker/setup-buildx-action@v1 
      - name: Login to Docker 
        # Запуск скрипта авторизации на Docker Hub
        uses: docker/login-action@v1 
        with:
          username: <имя-пользователя>
          password: <пароль-доступ-к-докер-хаб>
      - name: Push to Docker Hub
        # Пуш образа в Docker Hub 
        uses: docker/build-push-action@v2 
        with:
          push: true
          tags: <имя-пользователя>/<имя-репозитория>:latest
```

**Как хранить секреты в GitHub Actions**

На платформе GitHub Actions переменные окружения c токенами, паролями и другими приватными данными можно хранить в зашифрованном виде прямо в репозитории. 
В системе GitHub Actions такие переменные называют **secrets**, хранят их в разделе **Secrets**.

Создавать *secrets* можно только в собственных репозиториях:
* Перейдите в настройки репозитория **Settings**, выберите на панели слева **Secrets**, нажмите **New secret**:
* Сохраните переменные *DOCKER_USERNAME* и *DOCKER_PASSWORD* с необходимыми значениями: задайте имя секрета и его значение, затем нажмите Add secret

Обращаться к нему можно так:`${{ secrets.<ИМЯ_СЕКРЕТА> }}`

Теперь отредактируем:
```yaml
build_and_push_to_docker_hub:
    ...
    steps:
      ...
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      ...
```

## Настоящий workflow: deploy

Теперь нужно настроить автоматический деплой проекта на боевой сервер. Деплой должен происходить, только если тесты и обновление образа в Docker Hub прошли успешно.

**Подготовка сервера.**

Чтобы настроить сервер для работы с Docker, подключитесь к нему через консоль. Адрес сервера указывается по IP или доменному имени. 
Команда для подключения вводится в формате:
```
ssh <имя-пользователя>@<IP-адрес сервера>
```
Установите на свой сервер Docker:`sudo apt install docker.io`

**Подключение к удалённому серверу**

Для того чтобы запустить обновлённый проект на боевом сервере, сервер должен:
* скачать с Docker Hub обновлённый образ проекта;
* остановить и удалить запущенный контейнер с проектом;
* запустить контейнер из обновлённого образа.

Ваш боевой сервер получит команды не от вас, а от «незнакомого» сервера GitHub Actions. 
Чтобы боевой сервер позволил установить соединение, добавьте на сервер GitHub Actions *ssh-ключ* для подключения к боевому серверу.

Чтобы никто не мог получить доступ к вашему приватному ключу на GitHub Actions, сохраните ключ в Secrets.
* Скопируйте приватный ключ с компьютера, имеющего доступ к боевому серверу: `cat ~/.ssh/id_rsa`
* На GitHub Actions сохраните ключ в Secrets, в переменную `SSH_KEY`.

Помимо ключа для доступа к серверу нужны username и адрес хоста. Тоже сохраните их в Secrets:
* в secret-переменную `USER` сохраните имя пользователя для подключения к серверу;
* в secret-переменную `HOST` сохраните IP-адрес вашего сервера;
* если при создании ssh-ключа вы использовали фразу-пароль, то сохраните её в secret-переменную `PASSPHRASE`.

Добавьте в workflow ещё один job — в нём будут инструкции для скачивания на боевой сервер обновлённого образа и запуска контейнера:

```yaml
deploy:
    runs-on: ubuntu-latest
    needs: build_and_push_to_docker_hub
    steps:
    - name: executing remote ssh commands to deploy
      uses: appleboy/ssh-action@master
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USER }}
        key: ${{ secrets.SSH_KEY }}
        passphrase: ${{ secrets.PASSPHRASE }} # Если ваш ssh-ключ защищён фразой-паролем
        script: |
          # Выполняет pull образа с DockerHub
          sudo docker pull <имя-пользователя>/<имя-репозитория>
          #остановка всех контейнеров
          sudo docker stop $(sudo docker ps -a -q)
          sudo docker run --rm -d -p 5000:5000 <имя-пользователя>/<имя-репозитория>
```
По инструкции `docker run` из докер-образа будет собран и запущен контейнер на порте 5000 — порт указан в параметре `-p`. 
Параметр `-d` устанавливает, что контейнер будет запущен в фоновом режиме, а параметр `--rm` определяет, что при остановке контейнер будет автоматически удалён.

**Отправка отчёта**

Добавьте ещё один шаг в workflow:
```yaml
send_message:
  runs-on: ubuntu-latest
  needs: deploy
  steps:
  - name: send message
    uses: appleboy/telegram-action@master
    with:
      to: ${{ secrets.TELEGRAM_TO }}
      token: ${{ secrets.TELEGRAM_TOKEN }}
      message: ${{ github.workflow }} успешно выполнен!
```
Зайдите в Settings → Secrets в вашем репозитории и добавьте ещё две переменные:
* чтобы бот отправил сообщение именно вам, в переменной `TELEGRAM_TO` сохраните ID своего телеграм-аккаунта. Узнать свой ID можно у бота @userinfobot;
* в переменной `TELEGRAM_TOKEN` сохраните токен вашего бота. Получить этот токен можно у бота @BotFather.
