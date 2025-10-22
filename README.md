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

4. Поднимите контейнер с базой данных PostgreSQL
```powershell
docker-compose -f postgres.yaml up -d
```

5. Запустите django-deplpoyment.yaml
Перейдите в папку с манифестами Kubernetes:
```powershell
cd C:\project\k8s-test-django\kubernetes
```
Примените манифест:
```powershell
kubectl apply -f django-deplpoyment.yaml
```
Он создаст pod и сервис в вашем кластере.

6. Примените секреты:
```powershell
kubectl apply -f secret.yaml
```
*Подробнее про секреты читайте дальше.

7. Примените миграции:
```powershell
kubectl apply -f django-migrate.yaml
```

8. Создайте суперпользователя для входа в админку:
  - узнайте имя пода `django-deplpoyment`:
  ```powershell
  kubectl get pods
  ```
  - зайдите в консоль пода:
  ```powershell
  kubectl exec -it <имя_пода> -- bash
  ```
  - создайте пользователя:
  ```powershell
  python manage.py createsuperuser
  ```

9. Получите URL для доступа к сайту через Minikube:
```powershell
minikube service django-service --url
```
Откройте полученный URL в браузере.

10. Для автоматической очистки устаревших сессий нужно применить CronJob. Манифест находится в папке `kubernetes` и назывется `cronJob-clearsessions.yaml`, применить его можно командой:
```powershell
kubectl apply -f cronJob-clearsessions.yaml
```
Проверить что манифест применился можно командой:
```powershell
kubectl get cronjob
```
Задание настроено на автоматическое применение команды `clearsessions` первого числа каждого месяца в 00:00.\
Для изменения даты и времени необходимо изменить поле `schedule: "0 0 1 * *"` в файле `cronJob-clearsessions.yaml`.

Для выбора расписания используйте инструкцию ↓↓↓
```txt
# ┌───────────── minute (0 - 59)
# │ ┌───────────── hour (0 - 23)
# │ │ ┌───────────── day of the month (1 - 31)
# │ │ │ ┌───────────── month (1 - 12)
# │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
# │ │ │ │ │                                   OR sun, mon, tue, wed, thu, fri, sat
# │ │ │ │ │
# │ │ │ │ │
# * * * * *
```

Что бы вручную запустить задание, можно использовать команду:
```powershell
kubectl create job --from=cronjob/<cronjob-name> <job-name> -n <namespace-name>
``` 
например:
```powershell
kubectl create job --from=cronjob/cronjob-clearsessions django-clearsessions-test
```
Проверить что всё получилось можно командой:
```powershell
kubectl get jobs
```
`STATUS` должен быть `Complete`\
`COMPLETIONS` 1/1


#### Создание базы данных через Helm

- Подготовьте файл `values.yaml` с данными пользователя базы данных:
```yaml
global:
  postgresql:
    auth:
      postgresPassword: admin      # пароль суперпользователя postgres
      username: myuser             # имя пользователя
      password: mypassword         # пароль для этого пользователя
      database: mydatabase         # имя базы данных, которая будет создана
```

- Обновите `DATABASE_URL` в `secret.yaml` с учётом указанных данных:
```yaml
postgres://myuser:mypassword@postgres-db-postgresql:5432/mydatabase 
```

- Установите бд в ваш кластер с помощью Helm:
```powershell
helm install postgres-db oci://registry-1.docker.io/bitnamicharts/postgresql -f values.yaml
```

- Убедитесь, что поды запущены и база работает:
```powershell
kubectl get pods
```


#### Работа с секретами (Secret) в Kubernetes

В Kubernetes чувствительные данные, такие как SECRET_KEY, DATABASE_URL и другие параметры, можно хранить безопасно с помощью объекта Secret.

Создайте файл `secret.yaml` c таким содержанием:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: secret
type: Opaque
data:
  SECRET_KEY: REPLACE_WITH_BASE64
  DATABASE_URL: REPLACE_WITH_BASE64
  DEBUG: REPLACE_WITH_BASE64
  ALLOWED_HOSTS: REPLACE_WITH_BASE64
```

Замените `REPLACE_WITH_BASE64` на свои значение, предварительно закодируйте свои значение в `base64`, команда для кодирования:\
Для Linux/macOS. Используйте `Git Bash` если у вас Windows:
```bash
echo -n "12345qwerty" | base64
```
Результат:
```bash
MTIzNDVxd2VydHk=
```
О том какие значения использовать читайте ниже.

Для приминения секрета в кластере используйте команду:
```powershell
kubectl apply -f secret.yaml
```

Проверить успешное создание секрета можно командой:
```powershell
kubectl get secrets
```

В манифестах можно ссылаться на секрет такой структурой:
```yaml
envFrom:
  - secretRef:
      name: secret
```


### Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


### Настройка Ingress для Django в Minikube

Эта инструкция описывает, как настроить Ingress для проекта Django в Minikube с использованием NGINX Ingress Controller.

- Включите NGINX Ingress Controller:
```powershell
minikube addons enable ingress
```

- Проверьте работу:
```powershell
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

- Создайте Ingress манифест:
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "false"  # Отключяет перенаправление на https
spec:
  ingressClassName: nginx
  rules:
  - host: star-burger.test  # Имя вашего локального домена
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: django-service  # Имя вашего сервиса
            port:
              number: 80  # Порт сервиса
```

- Примените Ingress:
```powershell
kubectl apply -f django-ingress.yaml
```

- Если домен star-burger.test не зарегистрирован, для локальной разработки нужно указать в `hosts` что бы перенаправить с 127.0.0.1 на ваш домен "star-burger.test".\
файл `hosts` находится по такому пути: `C:\Windows\System32\drivers\etc`, добавьте туда запись:
```txt
127.0.0.1 star-burger.test
```

- Проверьте работу Ingress:
```powershell
kubectl get ingress
kubectl describe ingress django-ingress
```

- Откройте сайт в браузере `http://star-burger.test/`.
Иногда нужно открыть сайт в режиме инкогнито или очистить кэш браузера, иначе старые DNS-записи могут мешать.

### Как подготовить dev окружение yandex cloud

1. Получение SSL-сертификата:

Linux/macOS
```bash
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0655 ~/.postgresql/root.crt
```

Windows
```powershell
mkdir $HOME\.postgresql; curl.exe -o $HOME\.postgresql\root.crt https://storage.yandexcloud.net/cloud-certs/CA.pem
```

2. Создайте секрет с SSL-сертификатом для подключения к PostgreSQL:\
Перейдите в `Lens Desktop`:\
Перейдите в `Secrets` → `Create Secret`\
Name: postgres-ssl-cert\
Namespace: ваш_Namespace\
Data:\
Key: ca.crt\
Value: содержимое SSL-сертификата

3. Запуск тестового Pod
Примените манифест для создания тестового Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: название_пода
  namespace: ваш_Namespace
spec:
  containers:
    - name: название_контейнера
      image: postgres:13-alpine    # Образ postgres
      envFrom:
        - secretRef:
            name: postgres   # Название секрета с данными БД
      volumeMounts:
        - name: ssl-cert
          mountPath: /root/.postgresql/root.crt  # Куда сохранить сертификат внутри контейнера
          subPath: ca.crt   # Ключ из секрета созданного ранее
          readOnly: true
      command:
        - sh
        - -c
        - |
          chmod 0600 /root/.postgresql/root.crt
          sleep infinity
  volumes:
    - name: ssl-cert
      secret:
        secretName: postgres-ssl-cert
        items:
          - key: ca.crt
            path: ca.crt
```

3. Подключение к PostgreSQL\
После запуска Pod подключитесь к нему и проверьте соединение с БД:
```powershell
kubectl exec -n <ваш_Namespace> -it <название_пода> -- sh
```

Внутри контейнера подключиться к PostgreSQL:
```bash
psql "host=$host port=$port dbname=$name user=$username password=$password sslmode=verify-full sslrootcert=/root/.postgresql/root.crt"
```

4. Полезные команды для проверки подключения к БД:
```sql
-- Проверить версию PostgreSQL
SELECT version();

-- Проверить текущую базу данных
SELECT current_database();

-- Проверить текущего пользователя
SELECT current_user;

-- Посмотреть все базы данных
\l

-- Посмотреть все схемы
\dn
```