# Django Site

Докеризированный сайт на Django для экспериментов с Kubernetes.
Внутри контейнера Django приложение запускается с помощью Nginx Unit, не путать с Nginx. Сервер Nginx Unit выполняет сразу две функции: как веб-сервер он раздаёт файлы статики и медиа, а в роли сервера-приложений он запускает Python и Django. Таким образом Nginx Unit заменяет собой связку из двух сервисов Nginx и Gunicorn/uWSGI. [Подробнее про Nginx Unit](https://unit.nginx.org/).

## Как подготовить окружение к локальной разработке

Код в репозитории полностью докеризирован, поэтому для запуска приложения вам понадобится Docker. Инструкции по его установке ищите на официальных сайтах:
- [Get Started with Docker](https://www.docker.com/get-started/)
Вместе со свежей версией Docker к вам на компьютер автоматически будет установлен Docker Compose. Дальнейшие инструкции будут его активно использовать.

## Переменные окружения

Образ с Django считывает настройки из переменных окружения. Создайте файл .env
`SECRET_KEY` -- обязательная секретная настройка Django. Это соль для генерации хэшей. Значение может быть любым, важно лишь, чтобы оно никому не было известно. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#secret-key).
`DEBUG` -- настройка Django для включения отладочного режима. Принимает значения `TRUE` или `FALSE`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#std:setting-DEBUG).
`ALLOWED_HOSTS` -- настройка Django со списком разрешённых адресов. Если запрос прилетит на другой адрес, то сайт ответит ошибкой 400. Можно перечислить несколько адресов через запятую, например `127.0.0.1,192.168.0.1,site.test`. [Документация Django](https://docs.djangoproject.com/en/3.2/ref/settings/#allowed-hosts).
`DATABASE_URL` -- адрес для подключения к базе данных PostgreSQL. Другие СУБД сайт не поддерживает. [Формат записи](https://github.com/jacobian/dj-database-url#url-schema).

## Как запустить сайт для локальной разработки
Перед сборкой рекомендуется использовать команду для очистки кэш докера
``` shell
docker builder prune
```

Затем собрать образ из файла docker-compose.yaml
При сборке потянет файл Dockerfile из папки `backend_main_django`
Необходимо в этой же папке создать файл .env с описанием переменных окружения 
Они будут скопированы в итоговый образ ! 
```shell
$ docker compose up
```

Командой
```sh
docker image ls
```
в перечне образов должен появиться образ - `django_app`
Необходимо зайти в образ 
```sh
docker run --rm -it django_app /bin/bash
```
и проверить работоспособность команды ./manage.py

Запустите базу данных и сайт:
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
```
Running migrations:
No migrations to apply.

### Как добавить библиотеку в зависимости
В качестве менеджера пакетов для образа с Django используется pip с файлом requirements.txt. Для установки новой библиотеки достаточно прописать её в файл requirements.txt и запустить сборку докер-образа:

```sh
$ docker compose build web
```

Аналогичным образом можно удалять библиотеки из зависимостей.
<a name="env-variables"></a>

## Как установить Minikube
Установите [Minikube](https://kubernetes.io/ru/docs/tasks/tools/install-minikube/)

## Развернуть сайт в Minikube
Созданный образ докер добавить в к8s
```sh
minikube image load django_app:latest
```

#### Проверить список образов к8s можно используя команду
```sh
minikube image ls --format=table
```

#### Из загруженного образа в k8s нужно сделать deploy. Делать это через команду
```sh
kubectl apply -f deploy-django.yaml 
```
#### Дополнительно создаем раздел секретов для работы приложения из файла `.env` 
```sh
kubectl create secret generic my-secret --from-env-file=.env
```
#### Для выполнения миграций уже существующего деплоя - используйте следующий скрипт:
```sh
kubectl apply -f migrate.yaml  
```
#### Настройка и использование PostgreSQL в K8S
1. Создаем ресурс содержащий данные, которые используются в процессе развертывания.
СonfigMap. Подробнее про такой тип ресурса можно почитать [тут.](https://matthewpalmer.net/kubernetes-app-developer/articles/ultimate-configmap-guide-kubernetes.html)
С собержанием:
```sh
#Имя файла - postgres-configmap-01.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
  labels:
    app: postgres
data:
  #в нашем случае, как пример
  POSTGRES_DB: test_k8s
  POSTGRES_USER: test_k8s
  POSTGRES_PASSWORD: OwOtBep9Frut
```
 Сохраняем и запускаем его :
```sh
kubectl apply -f postgres-configmap-01.yaml
```

2. Создание тома постоянного хранилища данных для postgres размером 3Gb
```sh
#Create and Apply Persistent Storage Volume and Persistent Volume Claim
#Имя файла - postgres-storage-02.yaml
kind: PersistentVolume
apiVersion: v1
metadata:
  name: postgres-pv-volume
  labels:
    type: local
    app: postgres
spec:
  storageClassName: manual
  capacity:
    storage: 3Gi
  accessModes:
    - ReadWriteMany
  hostPath:
    path: "/home/zatomis/pg_data" <- укажите свой актуальный путь
---
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: postgres-pv-claim
  labels:
    app: postgres
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
```
Сохраняем и запускаем его :
```sh
kubectl apply -f postgres-storage-02.yaml
```

3. Создание и развертывания PostgreSQL. Пример версии 14.1
```sh
#Create and Apply PostgreSQL Deployment
#The deployment file contains configuration of the PostgreSQL deployment and provides specifications
#for the containers and volumes:
#Имя файла - postgres-deployment-03.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:14.1
          imagePullPolicy: "IfNotPresent"
          ports:
            - containerPort: 5432
          envFrom:
            - configMapRef:
                name: postgres-config
          volumeMounts:
            - mountPath: /var/lib/postgresql/data
              name: postgredb
      volumes:
        - name: postgredb
          persistentVolumeClaim:
            claimName: postgres-pv-claim
```
Сохраняем и запускаем его :
```sh
kubectl apply -f postgres-deployment-03.yaml
```

4. Создание службы PostgreSQL 
```sh
#Create and Apply PostgreSQL Service. Specify the service type and ports. 
#Имя файла - postgres-deployment-03.yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  type: NodePort
  ports:
   - port: 5432
  selector:
   app: postgres
```
Сохраняем и запускаем его :
```sh
kubectl apply -f postgres-service-04.yaml
```
5. Как подключиться к PostgreSQL
Воспользуйтесь командой для отображения всех созданных вами сервисов
```sh
kubectl get all
```
![img.png](img.png)
Нас будет интересовать номер пода PostgreSQL и ip адрес 
```sh
kubectl exec -it postgres-6f9f647bb9-7q8dg --  psql -h localhost -U test_k8s --password -p 5432 test_k8s
```
Для того чтобы всё запустить - необходимо в файле `.env` прописать ip адрес подключения к PostgreSQL
`DATABASE_URL=postgres://test_k8s:OwOtBep9Frut@10.109.136.234:5432/test_k8s`
После необходимо сделать миграцию и добавить пользователя

## Как установить сайт в локальной сети
Измените ваш локальный файл [hosts](https://help.reg.ru/support/dns-servery-i-nastroyka-zony/rabota-s-dns-serverami/fayl-hosts-gde-nakhoditsya-i-kak-yego-izmenit#0)
![img_1.png](img_1.png)

ip адрес Minikube можно получить командой
```sh
minikube ip 
```
Рабочий вариант сайта будет по адресу http://star-burger.test/ 
