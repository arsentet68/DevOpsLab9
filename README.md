# ФИТ-2024 НМ. Полынский Арсений. DevOps. ЛР8

## Установка minikube на Ubuntu

```
# Скачиваем и инсталлируем kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Скачиваем и инсталлируем minikube
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube_latest_amd64.deb
sudo dpkg -i minikube_latest_amd64.deb

# Добавляем себе права
sudo usermod -aG docker $USER && newgrp docker

# Отключаем своп
sudo swapoff -a

# Запускаем minikube в режиме однонодового кластера k8s
minikube start --vm-driver=docker
```

## Проверка minikube и k8s кластера

<img width="403" height="156" alt="image" src="https://github.com/user-attachments/assets/36ccd113-0b06-47a9-b970-fff86c42434c" />

<img width="614" height="139" alt="image" src="https://github.com/user-attachments/assets/7b989066-64ef-4dd4-bc2e-897ce4ddc0f8" />

## Создание структуры каталогов

<img width="306" height="242" alt="image" src="https://github.com/user-attachments/assets/9739c131-92da-4617-ba96-5a78f1eff410" />

## Модифицируем приложение Flask (app.py)

```
import time
import redis
import socket
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    return cache.incr('hits')

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World! I have been seen {} times. My name is {}'.format(count, socket.gethostname())
```

## Готовим образы контейнеров и делаем их доступными в кластере k8s

```
# Производим сборку образа из исходников прямо внутри minikube
minikube image build -t flask:v1 flask_redis/

# Загружаем готовый образ redis внутрь кластера
minikube image load redis:alpine

# Проверяем, что образы стали доступны внутри кластера
minikube image ls
```

## Готовим описания (манифесты) для каждого сервиса - Flask

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
  labels:
    app: flask-app
spec:
  replicas: 5
  selector:
    matchLabels:
      app: flask-app
      svc: front
  template:
    metadata:
      labels:
        app: flask-app
        svc: front
    spec:
      containers:
      - name: flask
        image: flask:v1
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 5000
        resources:
          limits:
            memory: "256Mi"
```

## Готовим описания (манифесты) для каждого сервиса - Flask-SERVICE

```
apiVersion: v1
kind: Service
metadata:
  name: service-devops
  labels:
    app: flask-app
spec:
  type: LoadBalancer
  selector:
    app: flask-app
    svc: front
  ports:
  - port: 8000
    targetPort: 5000
  externalIPs:
  - 10.0.2.15
```

## Готовим описания (манифесты) для каждого сервиса - Redis

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  labels:
    app: flask-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flask-app
  template:
    metadata:
      labels:
        app: flask-app
        svc: db
    spec:
      containers:
      - name: redis
        image: redis:alpine
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 6379
```

## Готовим описания (манифесты) для каждого сервиса - Redis-SERVICE

```
apiVersion: v1
kind: Service
metadata:
  name: redis
  labels:
    app: flask-app
spec:
  type: ClusterIP
  selector:
    app: flask-app
    svc: db
  ports:
  - port: 6379
    targetPort: 6379
```

## Применяем манифесты k8s

```
# применяем манифесты по отдельности файлами или весь каталог целиком
kubectl apply -f flask_redis_k8s/
```

<img width="556" height="108" alt="image" src="https://github.com/user-attachments/assets/607625db-257a-49f1-9120-1067c7a145a7" />

```
# проверяем статус развертывания реплик
kubectl get pods
```

<img width="577" height="176" alt="image" src="https://github.com/user-attachments/assets/6f44884c-2983-4233-a683-62ab6444b62e" />

```
# проверяем сервисы
kubectl get services
```

<img width="772" height="111" alt="image" src="https://github.com/user-attachments/assets/b3ee5dd6-ba04-4d29-88bb-cb7cae2bbdce" />

## Проверка

```
# Внутри вм в отдельном окне разрешаем проброс трафика вм внутрь minicube
minikube tunnel --bind-address 10.0.2.15
```

Проверяем в браузере http://localhost:8000

<img width="693" height="138" alt="image" src="https://github.com/user-attachments/assets/35cb9dc1-d399-4ad4-9d16-fd7e7a90be28" />

<img width="674" height="136" alt="image" src="https://github.com/user-attachments/assets/4390455a-6a6e-4e40-a94a-9ddebe112a6d" />

## Обновляем приложение на новую версию (Rollinng Update)

```
import time
import redis
import socket
from flask import Flask

app = Flask(__name__)
cache = redis.Redis(host='redis', port=6379)

def get_hit_count():
    return cache.incr('hits')

@app.route('/')
def hello():
    count = get_hit_count()
    return 'Hello World VERSION 5! I have been seen {} times. My name is {}'.format(count, socket.gethostname())
```

## Обновляем поды на новую версию (Rolling Update) - готовим новый образ

Пересобираем образ с новым тегом :v5 и делаем новый образ доступным в кластере minikube

<img width="1091" height="416" alt="image" src="https://github.com/user-attachments/assets/8665b412-a214-4a3a-972c-e65005095d36" />

## Обновляем поды на новую версию - патчим деплоймент

```
# Можно пропатчить описание деплоймента прямо в БД k8s:
kubectl set image deployments/flask-app flask=flask:v5
```

<img width="743" height="43" alt="image" src="https://github.com/user-attachments/assets/edd329e9-3f49-4a04-b83c-6383e7a40805" />

Наблюдаем, как пересоздаются реплики подов на новую версию

<img width="578" height="170" alt="image" src="https://github.com/user-attachments/assets/0dca5bcb-c141-4504-b9be-540f337dbb6e" />

<img width="788" height="135" alt="image" src="https://github.com/user-attachments/assets/5a1c4fed-a85f-4968-a39a-a19436ef258c" />

<img width="651" height="49" alt="image" src="https://github.com/user-attachments/assets/105f4437-a088-4b3f-b85d-eb02a2272b45" />

<img width="1074" height="879" alt="image" src="https://github.com/user-attachments/assets/e07db5ac-6a58-4336-b29b-2ae07b5e507e" />




