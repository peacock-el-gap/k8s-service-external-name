# Testing K8s ExternalName services

## Testing scenario
1) Run PostgreSQL database (docker-compose) on machine no. 1.
2) Run Camunda on K8s cluster (running on machine no. 2).
   1) Camunda will use PostgreSQL mentioned above.
   2) Caminda configuration will point directly to machine no. 2.
      1) Option 1 - using pure IP address of machine-1.
      2) Option 2 - using machine-1 name (we will use https://sslip.io DNS service)
3) Configure service type = ExternalName on K8s cluster mentioned above, that will point machine no. 1 name.
4) Reconfigure Camunda to point this service.
5) Test if it works.

## Initial --> final configuration
```
+------------------+              +------------------+
| machine_1        |              | machine_1        |
|   +------------+ |              |   +------------+ |
|   | PostgreSQL | |              |   | PostgreSQL | |
|   +--------^---+ |              |   +--------^---+ |
+------------|-----+              +------------|-----+
             |                                 |
+------------|--------+           +------------|--------+
| machine_2  |        |           | machine_2  |        |
|   +--------|------+ |           |   +--------|------+ |
|   | K8s    |      | |           |   | K8s    |      | |
|   |   +----+----+ | |   ===>    |   |   +----+----+ | |
|   |   | Camunda | | |           |   |   | service | | |
|   |   +---------+ | |           |   |   +----^----+ | |
|   +---------------+ |           |   |        |      | |
+---------------------+           |   |   +----+----+ | |
                                  |   |   | Camunda | | |
                                  |   |   +---------+ | |
                                  |   +---------------+ |
                                  +---------------------+
```

## PostgreSQL @ machine no. 1

### Start PostgreSQL on machine no. 1
```shell
# This script has to be executed in project root
docker-compose -f ./machine-1/docker-compose.yaml up -d
```

### Stop PostgreSQL on machine no. 1
```shell
# This script has to be executed in project root
docker-compose -f ./machine-1/docker-compose.yaml down
```

## Camunda on K8s cluster

### Create namespace (if necessary)
```shell
kubectl create namespace camunda
```
### Create secret (database user & password)
```shell
kubectl create secret generic \
  camunda-bpm-platform-postgresql-credentials \
  --from-literal=DB_USERNAME=postgres \
  --from-literal=DB_PASSWORD=postgres \
  --namespace=camunda
```

### Install service with type: ExternalName
```shell
# This script has to be executed in project root
kubectl -n camunda apply -f ./machine-2/postgres-external-name.yaml
```

### Install Camunda
If necessary, add Camunda helm repo.
```shell
helm repo add camunda https://helm.camunda.cloud
helm repo update
```
Install Camunda.
```shell
# This script has to be executed in project root
helm upgrade \
  --namespace camunda --create-namespace \
  --install camundademo camunda/camunda-bpm-platform \
  --values ./machine-2/camunda-values.yaml
```

### Uninstall Camunda (if you want)
```shell
helm --namespac camunda uninstall camundademo
```



