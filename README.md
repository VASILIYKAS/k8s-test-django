# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:

- [Get Started with Docker](https://www.docker.com/get-started/)

Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Как запустить сайт для локальной разработки

Запустите базу данных и сайт:

```shell
$ docker compose up
```

В новом терминале, не выключая сайт, запустите несколько команд:

```shell
$ docker compose run --rm web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker compose run --rm web ./manage.py createsuperuser  # создаём в БД учётку суперпользователя
```

Готово. Сайт будет доступен по адресу [http://127.0.0.1:8080](http://127.0.0.1:8080). Вход в админку находится по адресу [http://127.0.0.1:8000/admin/](http://127.0.0.1:8000/admin/).

## Как вести разработку

Все файлы с кодом django смонтированы внутрь докер-контейнера, чтобы Nginx Unit сразу видел изменения в коде и не требовал постоянно пересборки докер-образа -- достаточно перезапустить сервисы Docker Compose.

### Как обновить приложение из основного репозитория

Чтобы обновить приложение до последней версии подтяните код из центрального окружения и пересоберите докер-образы:

``` shell
$ git pull
$ docker compose build
```

После обновлении кода из репозитория стоит также обновить и схему БД. Вместе с коммитом могли прилететь новые миграции схемы БД, и без них код не запустится.

Чтобы не гадать заведётся код или нет — запускайте при каждом обновлении команду `migrate`. Если найдутся свежие миграции, то команда их применит:

```shell
$ docker compose run --rm web ./manage.py migrate
…
Running migrations:
  No migrations to apply.
```

### Как добавить библиотеку в зависимости

В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.


### Как запустить кластер Kubernetes и деплой Django-приложения

#### Подготовка и установка
1. [Скачать](https://kubernetes.io/releases/download/#binaries) `kubectl`
Для Windows необходим файл "windows	amd64	kubectl.exe"

2. [Скачать](https://github.com/kubernetes/minikube/releases/tag/v1.37.0) `minikube`
Для Windows необходим файл "minikube-windows-amd64.exe"
Переименуйте его в `minikube.exe`

3. Для работы Minikube нужен “виртуализатор”, чтобы создать виртуальную машину в котором будет работать кластер Kubernetes.
Можно использовать `VirtualBox` или другой гипервизор. 
Рекомендую использовать `Docker Desktop`. В этом случае кластер запускается в контейнерах Docker, а не в полноценной VM\
[Скачайте](https://www.docker.com/products/docker-desktop/) и установите `Docker-desktop`.

4. Поместите файлы `kubectl.exe` и `minikube.exe` в папку на локальном диске, например `C:\kubernetes`
5. Добавьте путь в переменные среды:
  - Нажмите Win + S и найди “Переменные среды”.
  - Во вкладке "Дополнительно" выберите "Переменные среды...".
  - Найдите переменную `Path` и нажмите "изменить".
  - Нажмите "Создать" и укажите путь к папке с файлами `kubectl.exe` и `minikube.exe`\
  Например: `C:\kubernetes`

6. Проверьте работу minikube и kubectl с помощью команд:
```powershell
minikube version
kubectl version
```
Результат успешной команды:
```powershell
minikube version: v1.37.0
commit: 65318f4cfff9c12cc87ec9eb8f4cdd57b25047f3

Client Version: v1.32.2
Kustomize Version: v5.5.0
Server Version: v1.34.0
```

#### Запуск кластера
1. Откройте терминал и запустите кластер, команда:
```powershell
minikube start
```
*Docker-desktop должен быть запущен.

2. Скачайте проект и перейдите в папку с проектом:
```powershell
cd C:\project
git clone https://github.com/VASILIYKAS/k8s-test-django.git
```
*Замените диск "C:" и "project" на удобное для вас название папки и путь

3. Сборка образа Django\
Перейдите в директорию с Dockerfile:
```powershell
cd C:\project\k8s-test-django\backend_main_django
```
Соберите образ для Minikube:
```powershell
minikube image build -t django_app:latest .
```

4. Создайте Pod с Django\
Перейдите в папку с манифестами Kubernetes:
```powershell
cd C:\project\k8s-test-django\kubernetes
```
Примените манифест пода:
```powershell
kubectl apply -f django-pod.yaml
```

5. Поднимите контейнер с базой данных PostgreSQL
```powershell
docker-compose -f postgres.yaml up -d
```

6. Примените миграции:
```powershell
kubectl exec -it django -- python manage.py migrate
```

7. Создайте суперпользователя для входа в админку:
```powershell
kubectl exec -it django -- python manage.py createsuperuser
```

8. Примените секреты:
```powershell
kubectl apply -f secret.yaml
```
*Подробнее про секреты читайте дальше.

9. Создайте сервис для Django:
```powershell
kubectl apply -f django-service.yaml
```

10. Получите URL для доступа к сайту через Minikube:
```powershell
minikube service django-service --url
```
Откройте полученный URL в браузере.

#### Работа с секретами (Secret) в Kubernetes

В Kubernetes чувствительные данные, такие как SECRET_KEY, DATABASE_URL и другие параметры, можно хранить безопасно с помощью объекта Secret.

Создайте файл `secret.yaml`:

### Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).
