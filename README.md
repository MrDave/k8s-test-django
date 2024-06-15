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

<a name="env-variables"></a>
## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).


## Using Kubernetes and Minikube

Start Minikube and enable Ingress addon:
```sh
minikube start
minikube addons enable ingress
```

Configure k8s Secret - see [environmental variables section](#environmental-variables-in-kubernetes).

Apply `djangoapp.yaml` manifest file:
```sh
kubectl apply -f djangoapp.yaml
```

Verify that deployment, service and pod are launched:
```sh
kubectl get all
```
You'll see a list similar to this:
```
$ kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/djangoapp-6cf7cf8889-v4ckd   1/1     Running   0          3h10m

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/djangoapp-svc   ClusterIP   10.111.123.164   <none>        80/TCP    3h42m
service/kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP   54d

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/djangoapp   1/1     1            1           3h42m
```

### Environmental variables in Kubernetes
Kubernetes uses `Secret` to get environmental variables needed for Django to work.
Create a `django-secret.yaml` file (or copy/edit `example-django-secret.yaml`) and write down your vars as follows:
```YAML
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
stringData:
  DEBUG: "False"
  SECRET_KEY: "secret_key"
  DATABASE_URL: "database_url"
  ALLOWED_HOSTS: "allowed_hosts"
  
```
Then use `kubectl apply` to create `Secret`:
```sh
kubectl apply -f django-secret.yaml
```
To verify that the Secret is successfully configured, run:
```sh
kubectl get secrets
```
You will see your created secret with DATA reflecting the amount of variables that was set:
```
$ kubectl get secrets
NAME            TYPE     DATA   AGE
django-secret   Opaque   5      54d
```

If you add/modify env variables, edit the file and `kubectl apply` it once again.

### Accessing website
An Ingress is set up to access the Django site at http://star-burger.test. 
Add the minikube's IP to /etc/hosts:
```
$ minikube ip
192.168.59.101
```
`/etc/hosts`:
```
# previous rows

192.168.59.101 star-burger.test
```

Start `minikube tunnel` in a separate terminal window.

### Clearing Django sessions
To have Django sessions cleared regularly, apply `django-clearsessions.yaml` manifest file:
```sh
kubectl apply -f django-clearsessions.yaml
```
This will create a CronJob that will run a Job doing Django's `manage.py clearsession` once a month.