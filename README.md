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

```shell
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

`<a name="env-variables"></a>`

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Запуск в minikube

Установите [minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/) и [ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)

Далее нужно создать файл с секретами `django-secret.yml`:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
data:
 DATABASE_URL: <Формат записи:postgresql://user:password@url:5432/db >
  SECRET_KEY: <Секретный ключ Django>
  DEBUG: <Режим отладки>
  ALLOWED_HOSTS: <star-burger.test>
```

Все необходимо записывать в кодировку base64, чтобы закодировать данные используйте:

```yaml
echo test | base64
```

Декодировать можно:

```yaml
echo dGVzdAo= | base64 -d
```

### Запуск postgres через helm

```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
```

```
helm install pg-django bitnami/postgresql
--version <VERSION>
--set auth.postgresPassword=<YOUR_ROOT_PASSWORD>
--set auth.password=<YOUR_PASSWORD>
--set auth.username=<YOUR_USERNAME>
--set auth.database=<YOUR_DATABASE>
```

Потом узнаем адрес хоста, чтобы добавить его в секреты

```
kubectl get svc | grep pg-django
```

Дальее добавляем доменное имя к адресу `minicube ip`

`sudo nano /etc/host`

```
192.168.59.104  star-burger.test
```

Выполняем комманду

```
kubectl apply -f <Название файла>

```

Чтобы прмиенить миграции запустите файл `django-migrations.yml`

## Запуск dev окружения

Файлы конфигурации для запуска сервиса на Yandex Cloud хранятся в папке yc-sirius/edu-sweet-sanderson

Сначала нужно подключиться к кластеру Yandex Cloud. Необходимо:

1. [Инициализировать интерфейс командной строки](https://yandex.cloud/ru/docs/cli/quickstart#install)
2. [Добавить учетную данные](https://yandex.cloud/ru/docs/managed-kubernetes/operations/connect/#kubectl-connect)

```
yc managed-kubernetes cluster get-credentials --id <cluster-id> --external
```

- Убедитесь, что настроен доступ к Kubernetes кластеру.
- Примените Service и Pod
- Вы увидите приветственную страницу nginx.

### Как подготовить dev окружение:

Необходимо получить [SSL-сертификат.](https://yandex.cloud/ru/docs/managed-postgresql/operations/connect)

```
mkdir -p ~/.postgresql && \
wget "https://storage.yandexcloud.net/cloud-certs/CA.pem" \
     --output-document ~/.postgresql/root.crt && \
chmod 0655 ~/.postgresql/root.crt
```

Найти его можно в `~/.postgresql/root.crt` - его необходимо закодировать в base64 и создать secret

Пример:

```
apiVersion: v1
kind: Secret
metadata:
  name: pg-cert <Имя секрета>
  namespace: <Ваш namespace>
data:
  root.crt: <Ваш сертификат в кодировке base64>

```

Теперь можно подключится к поду:

```
kubectl exec -it <Например ubuntu> -- /bin/bash
```

И после чего войти в psql

```
psql "host=rc1b-qit*****0k.mdb.yandexcloud.net \
    port=6432 \
    sslmode=require \ <Параметр указывает на то, что соединение обязательно должно использовать SSL>
    dbname= <ваши данные>\
    user= <ваши данные>\
    password=<ваши данные>
```

### Запуск django приложения

Сначала запускаем деплоймент с самого django и секрет с ssl

```
kubectl apply -f postgres-secret.yml
```

```
kubectl apply -f django-deployment.yml
```

Даллее разворчиваем секрет и сервис

```
kubectl apply -f django-secret.yml
```

```
kubectl apply -f django-service.yml
```

Если запускаете проект впервые, то также понадобиться отмигрировать базу данных и поставить отчистку сессий на таймер:

```
kubectl apply -f django-clearsessions.yml
```

```
kubectl apply -f django-migrations.yml
```

[Пример сайта](https://edu-sweet-sanderson.sirius-k8s.dvmn.org/)

[Сслыка на инфраструктуру](https://sirius-env-registry.website.yandexcloud.net/edu-sweet-sanderson.html)
