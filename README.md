# Django site

Докеризированный сайт на Django для экспериментов с Kubernetes.

Внутри конейнера Django запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как запустить dev-версию

Запустите базу данных и сайт:

```shell-session
$ docker-compose up
```

В новом терминале не выключая сайт запустите команды для настройки базы данных:

```shell-session
$ docker-compose run web ./manage.py migrate  # создаём/обновляем таблицы в БД
$ docker-compose run web ./manage.py createsuperuser
```

Для тонкой настройки Docker Compose используйте переменные окружения. Их названия отличаются от тех, что задаёт docker-образа, сделано это чтобы избежать конфликта имён. Внутри docker-compose.yaml настраиваются сразу несколько образов, у каждого свои переменные окружения, и поэтому их названия могут случайно пересечься. Чтобы не было конфликтов к названиям переменных окружения добавлены префиксы по названию сервиса. Список доступных переменных можно найти внутри файла [`docker-compose.yml`](./docker-compose.yml).

## Переменные окружения

Образ с Django считывает настройки из переменных окружения:

`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).

`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).

`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).

`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Поднимаем приложение в кластере minikube

## minikube 

Для работы понадобятся: 
- [Kubectl](https://kubernetes.io/ru/docs/tasks/tools/install-kubectl/)
- [minikube](https://minikube.sigs.k8s.io/docs/start/)
- [VirtualBox](https://www.virtualbox.org) - в случае Linux\Windows\MacOS.
- [DockerDesktop](https://www.docker.com/products/docker-desktop/) - MacOS для Mac на M1.
- [Доменное имя](https://www.reg.ru) - если собираетесь поднимать еще и в облаке.
- [Учетная запись в VKcloud](https://mcs.mail.ru/app) - если собираетесь поднимать еще и в облаке.

### Переменные окружения.

<details>
<summary>
Раскрыть
</summary>


1) В папке kubernetes отредактируйте файл `configmap_example.yaml` и переименуйте его в `configmap.yaml`.
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: django-env
data:
  SECRET_KEY: "verysecret"
  DEBUG: "False"
  DATABASE_URL: "read README.md"
  ALLOWED_HOSTS: 123.123.123.123,dns.dns.ru,minkube.ip
```

</details>

### Конфигурируем манифест для Postgres.

<details>
<summary>
Раскрыть
</summary>

Откройте файл postgres-deployment.yaml
```yaml
...
spec:
      containers:
        - name: pg
          image: postgres:14
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 5432
              name: pg
          env:
            - name: POSTGRES_PASSWORD
              value: mypostgresspassword1
...
```
Можете поменять `POSTGRES_PASSWORD` на какой-то другой или создать `ConfigMap` манифест для хранения пароля Postgres (хотя можно использовать текущий `ConfigMap`). Если будете менять порт, то меняйте его и в `Service` манифесте в этом же файле ниже.
</details>


### Самое время заняться [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/) манифестом.

<details>
<summary>
Раскрыть
</summary>

Откройте файл `django-ingress.yaml`.

Настройка, убирающая автоматический редирект на https:
```yaml
nginx.ingress.kubernetes.io/rewrite-target: /
nginx.ingress.kubernetes.io/ssl-redirect: "false"
```

Вместо `firstapp.django-development.ru` необходимо указать ваше доменное имя, указанный в hosts файле.    
В качестве ip для доменного имени необходимо указать ip вашего minikube кластера.
```shell
minikube ip
```

Путь к файлу hosts:

- Windows10 - `C:\Windows\System32\drivers\etc\hosts`
- Linux - `/etc/hosts`
- Mac OS X - `/private/etc/hosts`

```yaml
rules:
  - host: firstapp.django-development.ru
```
Рассмотрим последнюю часть файла:

`path` - путь, при переходе на который произойдет редирект на порт `80` приложения `django-web`.

```yaml
paths:
  - path: /
    pathType: Prefix
    backend:
      service:
        name: django-web
        port:
          number: 80
```

</details>

### Запускаем приложение!


<details>
<summary>
Раскрыть
</summary>

3) Примените все манифесты командой:
```shell
kubectl -n django-app apply -f kubernetes/ 
```
`-n django-app` - позволяет применить манифесты в [namespace](https://kubernetes.io/ru/docs/concepts/overview/working-with-objects/namespaces/) `django-app`.

После этого можно проверить работоспособность подов. Выполните команду:
```shell
kubectl -n django-app get pods
```
Ура-ура! Кажется, все завелось!
```
NAME                                          READY   STATUS      RESTARTS   AGE
django-clearsessions-cron-27544328--1-v55f9   0/1     Completed   0          13s
django-web-775fcb56fc-ffg5t                   1/1     Running     0          3m18s
migrations--1-5rjgh                           0/1     Completed   0          2m24s
pg-5558587fb6-hz8t9                           1/1     Running     0          3m16s
```
Вы видите четыре пода: 
- приложение джанго `django-web-775fcb56fc-ffg5t`
- под для postgres `pg-5558587fb6-hz8t9`
- job, запускающий `миграции migrations--1-5rjgh`
- cronjob для очистки сессий `django-clearsessions-cron-27544328--1-v55f9`

После этого можно проверить работоспособность [сервисов](https://kubernetes.io/docs/concepts/services-networking/service/). Выполните команду:
```shell
kubectl -n django-app get svc
```
Как и ожидалось, сервисов два: для `postgres` и для `django-app`.
```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
django-web   ClusterIP   10.254.155.3    <none>        80/TCP     6h47m
pg           ClusterIP   10.254.152.11   <none>        5432/TCP   6h47m
```

После этого можно проверить работоспособность [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/). Выполните команду:
```shell
kubectl -n django-app get svc
```
```
NAME         CLASS    HOSTS                            ADDRESS         PORTS   AGE
django-web   <none>   firstapp.django-development.ru   89.208.208.22   80      6h47m
```
`HOSTS` и `ADDRESS` должны быть соответственны доменному имени и ip из файла `/etc/hosts`
</details>

### Создаем администратора и проверяем работоспособность


<details>
<summary>
Раскрыть
</summary>

Для того, чтобы зайти в "админку", необходимо создать профиль администратора.    

Выполните команду:
```shell
kubectl -n django-app get pods
```
```
NAME                                          READY   STATUS      RESTARTS   AGE
django-web-775fcb56fc-ffg5t                   1/1     Running     0          3m18s
....
```
Затем напрямую подключитесь к поду `django-web-775fcb56fc-ffg5t`
```shell
kubectl -it exec -n django-app django-web-775fcb56fc-ffg5t -- sh
```
Позравляю! Вы "попали" в под. Как там внутри? Не тесно?
Самое время создать суперпользователя привычной командой
Затем напрямую подключитесь к поду `django-web-775fcb56fc-ffg5t`
```shell
python manage.py createsuperuser
```

</details>

### Поключаем Helm

<details>
<summary>
Раскрыть
</summary>

Хранить данные в Posgres, который "бежит" в поде не очень безопасно. Поэтому для безопасности будем использовать `helm`.

Для начала, "убейте" Postges и сервис, выполнив команду:

```shell
kubectl delete -n django-app -d postgres-deployment.yaml
```

- Установите `helm`. [по инструкции](https://helm.sh/docs/intro/install/)
Выполните следующие команды для добавления необходимого чарта и его установки:

```shell
$ helm repo add bitnami https://charts.bitnami.com/bitnami
$ helm install <NAME> bitnami/postgresql
```

По инструкции подключите базу к кластеру, а затем создайте новую таблицу и пользователя:

```
CREATE DATABASE {DB_NAME};
CREATE USER {DB_USER} WITH ENCRYPTED PASSWORD {PASSWORD};
GRANT ALL PRIVILEGES ON DATABASE {DB_NAME} TO {DB_USER};
```

Поменяйте строку `DATABASE_URL` в `configmap.yaml` на следующую:  
`DATABASE_URL` : `postgres://DB_USER:PASSWORD@<NAME>-postgresql:5432/DB_NAME`

Заново подключитесь к поду и создайте суперпользователя Django (теперь понятно, почему БД в поде - это не безопасно?)

</details>

### Заготовка для описания деплоймента в облаке VK


<details>
<summary>
Раскрыть
</summary>

```shell
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
```

```shell
curl -vk --resolve www.djangodev.dev:443:89.208.208.22 www.djangodev.dev www.djangodev.dev
``` 
</details>
