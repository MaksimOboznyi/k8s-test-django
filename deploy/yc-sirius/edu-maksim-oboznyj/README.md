# Подготовка dev-окружения

## Создание SSL Secret для PostgreSQL

Скачать сертификат:

```bash
mkdir -p ~/.postgresql

wget https://storage.yandexcloud.net/cloud-certs/CA.pem \
  --output-document ~/.postgresql/root.crt
```

Создать Secret в Kubernetes:

```bash
kubectl create secret generic postgres-ssl-cert \
  -n edu-maksim-oboznyj \
  --from-file=root.crt=root.crt
```

Проверить создание:

```bash
kubectl get secret postgres-ssl-cert -n edu-maksim-oboznyj
```

## Создание тестового Pod

```bash
kubectl apply -f deploy/yc-sirius/edu-maksim-oboznyj/psql-test-pod.yaml
```

Проверка сертификата:

```bash
kubectl exec -it psql-test -n edu-maksim-oboznyj -- ls -l /root/.postgresql
```

# Сборка и публикация Docker-образа

Получить хэш текущего коммита:

```bash
git rev-parse --short HEAD
```

Собрать Docker-образ:

```bash
docker build -t maksimoboznyi/k8s-test-django:<git-hash> ./backend_main_django
```

Загрузить образ в Docker Hub:

```bash
docker push maksimoboznyi/k8s-test-django:<git-hash>
```

Скачать образ из Docker Hub:

```bash
docker pull maksimoboznyi/k8s-test-django:<git-hash>
```