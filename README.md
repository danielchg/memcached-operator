# Basic Tutorial Operator SDK

## Initialize

```bash
mkdir memcached-operator
cd memcached-operator
operator-sdk init --domain danielchavero --repo github.com/danielchg/memcached-operator
```

## Add API and Controller

```bash
operator-sdk create api --group cache --version v1alpha1 --kind Memcached --resource --controller
```

## Build Docker image

```bash
docker login
make docker-build docker-push IMG="danielchavero/memcached-operator:v0.0.1"
```

## Deploy Operator

```bash
make deploy IMG="danielchavero/memcached-operator:v0.0.1"
```
