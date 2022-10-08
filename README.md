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