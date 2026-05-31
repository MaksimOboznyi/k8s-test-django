# K8s Test Django

Учебный проект для запуска Django-сайта в Kubernetes через Minikube.

В проекте используются:

- Docker Compose для запуска базы данных PostgreSQL
- Minikube для локального Kubernetes-кластера
- Kubernetes Deployment для запуска Django
- Kubernetes Service для доступа к сайту
- Kubernetes Secret для хранения секретных настроек

---

## Запуск базы данных

База данных запускается снаружи Kubernetes-кластера через Docker Compose.

Создайте файл:

```text
docker-compose.override.yml
```

Содержимое файла:

```yaml
services:
  db:
    ports:
      - "5432:5432"
```

Запустите базу данных:

```bash
docker compose up -d db
```

Проверьте, что база работает:

```bash
docker compose ps
```

В колонке `PORTS` должно быть:

```text
0.0.0.0:5432->5432/tcp
```

---

## Запуск Minikube

Запустите Minikube с подходящим драйвером.

Если используется Docker:

```bash
minikube start --driver=docker
```

Если используется VirtualBox:

```bash
minikube start --driver=virtualbox
```

Проверьте кластер:

```bash
kubectl get nodes
```

Ожидаемый результат:

```text
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   ...
```

---

## Сборка Docker-образа Django

Собирать образ нужно из корня проекта:

```bash
minikube image build -t django_app:latest backend_main_django
```

Проверьте, что образ появился в Minikube:

```bash
minikube image ls
```

В списке должен быть образ:

```text
docker.io/library/django_app:latest
```

---

## Kubernetes Secret

Для запуска Django в Kubernetes нужно создать объект `Secret`.

Создайте файл:

```text
kubernetes/django-secret.yaml
```

Пример содержимого:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: django-secret
type: Opaque
stringData:
  SECRET_KEY: your-secret-key
  DATABASE_URL: postgres://user:password@host:5432/database
```

Для локального запуска с базой данных из Docker Compose значение `DATABASE_URL` может выглядеть так:

```text
postgres://test_k8s:OwOtBep9Frut@host.minikube.internal:5432/test_k8s
```

Примените Secret в кластере:

```bash
kubectl apply -f kubernetes/django-secret.yaml
```

Проверьте, что Secret появился:

```bash
kubectl get secrets
```

В списке должен быть объект:

```text
django-secret
```

Файл `kubernetes/django-secret.yaml` содержит секретные данные, поэтому его нельзя коммитить.

Добавьте его в `.gitignore`:

```gitignore
kubernetes/django-secret.yaml
```

---

## Deployment

Файл:

```text
kubernetes/django-deployment.yaml
```

Содержимое:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django
spec:
  replicas: 1
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
        - name: django
          image: django_app:latest
          imagePullPolicy: Never
          ports:
            - containerPort: 80
          env:
            - name: SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: django-secret
                  key: SECRET_KEY

            - name: DEBUG
              value: "False"

            - name: ALLOWED_HOSTS
              value: "*"

            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: django-secret
                  key: DATABASE_URL
```

Примените Deployment:

```bash
kubectl apply -f kubernetes/django-deployment.yaml
```

Проверьте Deployment:

```bash
kubectl get deployments
```

Ожидаемый результат:

```text
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
django   1/1     1            1           ...
```

Проверьте Pod:

```bash
kubectl get pods
```

Ожидаемый результат:

```text
django-xxxxxxxxxx-xxxxx   1/1   Running   0   ...
```

---

## Service

Файл:

```text
kubernetes/django-service.yaml
```

Содержимое:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: django
spec:
  type: ClusterIP
  selector:
    app: django
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Примените Service:

```bash
kubectl apply -f kubernetes/django-service.yaml
```

Проверьте Service:

```bash
kubectl get service
```

Ожидаемый результат:

```text
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
django       ClusterIP   ...             <none>        80/TCP    ...
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   ...

---

## Ingress

Для доступа к сайту по домену используется Kubernetes Ingress.

Включите Ingress в Minikube:

```bash
minikube addons enable ingress
```

Проверьте, что Ingress Controller запущен:

```bash
kubectl get pods -n ingress-nginx
```

Файл:

```text
kubernetes/django-ingress.yaml
```

Содержимое:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django
spec:
  ingressClassName: nginx
  rules:
    - host: star-burger.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django
                port:
                  number: 80
```

Примените Ingress:

```bash
kubectl apply -f kubernetes/django-ingress.yaml
```

Проверьте Ingress:

```bash
kubectl get ingress
```

Добавьте домен в `/etc/hosts`:

```text
127.0.0.1 star-burger.test
```

Если используется Minikube с Docker-драйвером, запустите туннель:

```bash
minikube tunnel
```

Терминал с `minikube tunnel` нужно оставить открытым.

После этого сайт будет доступен по адресу:

```text
http://star-burger.test
```

Админка Django:

```text
http://star-burger.test/admin/
```

---

## Regular session cleanup

Для регулярного удаления устаревших Django-сессий используется Kubernetes CronJob.

Файл:

```text
kubernetes/django-clearsessions-cronjob.yaml
```

CronJob запускает команду:

```bash
./manage.py clearsessions
```

Применить CronJob:

```bash
kubectl apply -f kubernetes/django-clearsessions-cronjob.yaml
```

Проверить CronJob:

```bash
kubectl get cronjobs
```

Чтобы вручную проверить работу CronJob, можно создать Job:

```bash
kubectl create job django-clearsessions-once --from=cronjob/django-clearsessions
```

Проверить Job:

```bash
kubectl get jobs
```

Ожидаемый результат:

```text
django-clearsessions-once   1/1
```


---

## Запуск сайта

Полный порядок запуска:

```bash
docker compose up -d db
minikube image build -t django_app:latest backend_main_django
kubectl apply -f kubernetes/django-secret.yaml
kubectl apply -f kubernetes/django-deployment.yaml
kubectl apply -f kubernetes/django-service.yaml
kubectl apply -f kubernetes/django-ingress.yaml
```

Проверьте состояние объектов:

```bash
kubectl get secrets
kubectl get deployments
kubectl get pods
kubectl get service
kubectl get ingress
```

Сайт доступен по адресу:

```text
http://star-burger.test
```

Админка Django:

```text
http://star-burger.test/admin/
```

---

## Проверка Django shell

Получите имя Pod:

```bash
kubectl get pods
```

Зайдите в Django shell:

```bash
kubectl exec -it POD_NAME -- ./manage.py shell
```

Проверьте подключение к базе данных:

```python
from django.contrib.auth.models import User
User.objects.all()
```

Выйти из shell:

```python
exit()
```

---

## Полезные команды

Посмотреть Pod'ы:

```bash
kubectl get pods
```

Посмотреть сервисы:

```bash
kubectl get svc
```

Посмотреть Deployment'ы:

```bash
kubectl get deployments
```

Посмотреть Secret'ы:

```bash
kubectl get secrets
```

Посмотреть логи Django:

```bash
kubectl logs deployment/django
```

Проверить обновление Deployment:

```bash
kubectl rollout status deployment/django
```

Удалить Deployment:

```bash
kubectl delete deployment django
```

Удалить Service:

```bash
kubectl delete service django
```

---

## Возможные проблемы

### База данных не запущена

Проверьте:

```bash
docker compose ps
```

Если база не запущена:

```bash
docker compose up -d db
```

---

### Docker daemon недоступен

Если появляется ошибка:

```text
Cannot connect to the Docker daemon
```

Запустите Docker Desktop и проверьте:

```bash
docker ps
```

---

### kubectl не подключается к кластеру

Если появляется ошибка:

```text
The connection to the server 127.0.0.1 was refused
```

Проверьте Minikube:

```bash
minikube status
```

Запустите Minikube заново:

```bash
minikube start --driver=docker
```

или:

```bash
minikube start --driver=virtualbox
```

---

### Server Error 500

Проверьте логи Django:

```bash
kubectl logs deployment/django
```

Проверьте базу данных:

```bash
docker compose ps
```

Если база не запущена:

```bash
docker compose up -d db
```

---

### Bad Request 400

Проблема может быть в `ALLOWED_HOSTS`.

Для учебного локального запуска можно использовать:

```yaml
- name: ALLOWED_HOSTS
  value: "*"
```

В настоящем production так делать нельзя.

---

## Git

Перед коммитом проверьте статус:

```bash
git status
```

В коммит должны попасть:

```text
.gitignore
kubernetes/django-deployment.yaml
kubernetes/django-service.yaml
kubernetes/django-ingress.yaml
```

В коммит не должен попасть:

```text
kubernetes/django-secret.yaml
```

Добавьте файлы:

```bash
git add README.md .gitignore kubernetes/django-deployment.yaml kubernetes/django-service.yaml kubernetes/django-ingress.yaml
```

Создайте коммит:

```bash
git commit -m "add k8s manifest files"
```
