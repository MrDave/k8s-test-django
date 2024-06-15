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

### Setting up PostgreSQL database
Here, Helm is used to install and set up database. Follow [installation guide](https://helm.sh/docs/intro/install/) if Helm is not yet installed.  
If that's the first time you use Helm, initialize a Helm Chart repo:
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
```

Afterwards, install PostgreSQL chart:
```sh
helm install bitnami/postgresql --generate-name
```
Example result:
```$ helm install bitnami/postgresql --generate-name
NAME: postgresql-1718468289
LAST DEPLOYED: Sat Jun 15 19:18:11 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES: ...
```
Here, the release is called `postgresql-1718468289`. Replace this name with your release name when running commands.  
The output will also tell you several commands which are used next. Copy them from the output for ease of execution. 

Export the password for `postgres`:
```sh
export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql-1718468289 -o jsonpath="{.data.postgres-password}" | base64 -d)
```
Connect to database:
```sh
kubectl run postgresql-1718468289-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql:16.3.0-debian-12-r14 --env="PGPASSWORD=$POSTGRES_PASSWORD" \
      --command -- psql --host postgresql-1718468289 -U postgres -d postgres -p 5432
```

In psql shell create new table and user for the Django project and grant privileges:
```SQL
CREATE DATABASE myproject;
CREATE USER myproject_user WITH PASSWORD 'myproject_database_password';
ALTER ROLE myproject_user SET client_encoding TO 'utf8';
ALTER ROLE myproject_user SET default_transaction_isolation TO 'read committed';
ALTER ROLE myproject_user SET timezone TO 'UTC';
GRANT ALL PRIVILEGES ON DATABASE myproject TO myproject_user;
ALTER DATABASE myproject OWNER TO myproject_user;
``` 
Then exit with `\q`

Lastly write correct credentials in `DATABASE_URL` in `django-secret.yaml`:
```
DATABASE_URL: "postgres://myproject_user:myproject_database_password@postgresql-1718468289.default.svc.cluster.local:5432/myproject"
```

Finally, proceed to migrating.

### Migrating database

To migrate database, apply `django-migrate.yaml` manifest file:
```sh
kubectl apply -f django-migrate.yaml
```

The results of migration (or lack of it if there were no changes) can be seen in Job's logs:
```
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, sessions
Running migrations:
  No migrations to apply.
```

### Clearing Django sessions
To have Django sessions cleared regularly, apply `django-clearsessions.yaml` manifest file:
```sh
kubectl apply -f django-clearsessions.yaml
```
This will create a CronJob that will run a Job doing Django's `manage.py clearsession` once a month.

Launch the Job manually if needed:
```sh
kubectl create job --from=cronjob/django-clearsessions <job-name>
```
