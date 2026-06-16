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