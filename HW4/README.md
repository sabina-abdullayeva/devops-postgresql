# Домашнее задание 4

## Структура

- `images` — исходники и Dockerfile для backend и frontend;
- `k8s` — обычные Kubernetes-манифесты;
- `helm/counter` — Helm-чарт приложения;
- `certs` — сертификат. Приватный ключ исключён из Git.

## Подготовка

```bash
kind create cluster --config kind-config.yaml

kubectl apply -f \
  https://kind.sigs.k8s.io/examples/ingress/deploy-ingress-nginx.yaml
kubectl wait -n ingress-nginx --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller --timeout=180s

kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
  --type merge \
  -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/os":"linux","ingress-ready":"true"}}}}}'

# ВМ имеет ограничение на число потоков.
kubectl patch configmap ingress-nginx-controller -n ingress-nginx \
  --type merge -p '{"data":{"worker-processes":"2"}}'
kubectl rollout restart deployment/ingress-nginx-controller -n ingress-nginx
kubectl rollout status deployment/ingress-nginx-controller -n ingress-nginx
```

Сборка локальных образов:

```bash
docker build -t counter-backend:local images/counter-backend
docker build -t counter-frontend:local images/counter-frontend

kind load docker-image counter-backend:local --name course
kind load docker-image counter-frontend:local --name course
```

Если приватного ключа нет, сертификат создаем заново:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout certs/counter.key \
  -out certs/counter.crt \
  -subj "/CN=counter.localtest.me" \
  -addext "subjectAltName=DNS:counter.localtest.me,DNS:counter-a.localtest.me,DNS:counter-b.localtest.me"
```

## Установка манифестов

Пароли и приватные ключи создаются командами и не хранятся в манифестах

```bash
kubectl create namespace counter

kubectl -n counter create secret tls counter-tls \
  --cert=certs/counter.crt \
  --key=certs/counter.key

kubectl -n counter create secret generic postgres-secret \
  --from-literal=password="$(openssl rand -base64 24)"

kubectl apply -n counter -f k8s/
kubectl get all -n counter
```

## 1. Приложение

Проверка HTTPS без `-k`:

```bash
curl --cacert certs/counter.crt https://counter.localtest.me
curl --cacert certs/counter.crt https://counter.localtest.me/api/counter
curl --cacert certs/counter.crt -X POST https://counter.localtest.me/api/counter
```

Backend имеет только `ClusterIP` Service. Снаружи он доступен через Ingress
только по пути `/api`.

Порт backend задан в ConfigMap и равен `8081`, поэтому образ пересобирать не
нужно:

```bash
kubectl get configmap counter-config -n counter -o yaml
kubectl get pod -n counter -l app=backend \
  -o jsonpath='{.items[0].spec.containers[0].ports[0].containerPort}'
```

Проверка readiness probe:

```bash
kubectl patch deployment backend -n counter --type=json \
  -p='[{"op":"replace","path":"/spec/template/spec/containers/0/readinessProbe/httpGet/path","value":"/not-ready"}]'

kubectl get pod -n counter -l app=backend
kubectl get endpointslice -n counter \
  -l kubernetes.io/service-name=backend \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{"\n"}{end}'

curl -4 --cacert certs/counter.crt \
  https://counter.localtest.me/api/counter

kubectl rollout undo deployment/backend -n counter
kubectl rollout status deployment/backend -n counter
```

Новый Pod с неправильным адресом readiness остаётся `0/1` и получает
`ready=false`. Старый готовый Pod остаётся в балансировке, поэтому сервис
продолжает отвечать без простоя.

Ресурсы и QoS:

```bash
kubectl get pod -n counter -l app=backend \
  -o jsonpath='{.items[0].status.qosClass}{"\n"}'
```

Получается класс `Guaranteed`: CPU и память имеют одинаковые `requests` и
`limits`. Это ограничивает контейнер и даёт планировщику точные значения.

Проверка безопасного обновления и отката:

```bash
kubectl annotate deployment/backend -n counter \
  kubernetes.io/change-cause="несуществующий образ" --overwrite
kubectl set image deployment/backend -n counter \
  backend=counter-backend:not-found
kubectl rollout status deployment/backend -n counter --timeout=30s

curl --cacert certs/counter.crt https://counter.localtest.me/api/counter
kubectl rollout history deployment/backend -n counter
kubectl rollout undo deployment/backend -n counter
kubectl rollout status deployment/backend -n counter
```

`maxUnavailable: 0` оставляет старый готовый Pod, пока новый не запустился.

### Ответы

1. При трёх бэкендах значение счётчика может прыгать, потому что каждый под
   хранит свой счётчик в памяти, поэтому общее состояние нужно перенести в PostgreSQL
   или другое внешнее хранилище
2. Переменные из ConfigMap читаются при запуске Pod, после изменения нужен
   перезапуск. Файлы из ConfigMap обновляются постепенно, если не подключены
   через `subPath`.
3. Выбран `Guaranteed`, потому что для CPU и памяти `requests` равны `limits`.
   Это просто и защищает кластер от одного слишком большого контейнера

Проверка нескольких экземпляров:

```bash
kubectl scale deployment/backend -n counter --replicas=3
for i in 1 2 3 4 5 6; do
  curl --cacert certs/counter.crt -X POST \
    https://counter.localtest.me/api/counter
  echo
done
kubectl scale deployment/backend -n counter --replicas=1
```

## 2. PostgreSQL

StatefulSet выбран из-за стабильного имени Pod и отдельного постоянного диска.

Проверка подключения по стабильному имени:

```bash
PASSWORD=$(kubectl get secret postgres-secret -n counter \
  -o jsonpath='{.data.password}' | base64 -d)

kubectl run psql -n counter --rm -it --restart=Never \
  --image=postgres:17-bookworm \
  --env="PGPASSWORD=$PASSWORD" -- \
  psql -h postgres-0.postgres -U counter -d counter \
  -c "SELECT * FROM counter;"
```

Проверка сохранения данных:

```bash
kubectl run psql -n counter --rm -it --restart=Never \
  --image=postgres:17-bookworm \
  --env="PGPASSWORD=$PASSWORD" -- \
  psql -h postgres-0.postgres -U counter -d counter \
  -c "UPDATE counter SET value=10 WHERE id=1;"

kubectl delete pod postgres-0 -n counter
kubectl wait -n counter --for=condition=ready pod/postgres-0 --timeout=180s

kubectl run psql -n counter --rm -it --restart=Never \
  --image=postgres:17-bookworm \
  --env="PGPASSWORD=$PASSWORD" -- \
  psql -h postgres-0.postgres -U counter -d counter \
  -c "SELECT * FROM counter;"

kubectl delete statefulset postgres -n counter
kubectl wait -n counter --for=delete pod/postgres-0 --timeout=120s
kubectl get pvc -n counter

kubectl apply -n counter -f k8s/postgres.yaml
kubectl wait -n counter --for=condition=ready pod/postgres-0 --timeout=180s
```

Данные лежат в PVC `data-postgres-0`. При пересоздании StatefulSet PVC
остаётся. При удалении namespace PVC удалится, а стандартный локальный
StorageClass удалит и его данные.

## 3. Эксплуатация

```bash
kubectl get nodes
kubectl get daemonset,pod -n counter -l app=node-exporter -o wide
```

Проверка прав автоматики:

```bash
kubectl auth can-i update deployment/frontend -n counter \
  --subresource=scale \
  --as=system:serviceaccount:counter:frontend-scaler

kubectl auth can-i delete pods -n counter \
  --as=system:serviceaccount:counter:frontend-scaler
```

Первый ответ должен быть `yes`, второй — `no`.

Ручной запуск вне расписания:

```bash
kubectl create job -n counter --from=cronjob/frontend-scale-up scale-now
kubectl wait -n counter --for=condition=complete job/scale-now --timeout=120s
kubectl get deployment frontend -n counter
kubectl logs -n counter job/scale-now
```

CronJob использует часовой пояс `Europe/Moscow`: увеличение в 09:00,
уменьшение в 18:00.

## 4. Helm

Две установки используют разные namespace, имена ресурсов и адреса:

```bash
kubectl create namespace counter-a
kubectl create namespace counter-b

kubectl create secret tls counter-tls -n counter-a \
  --cert=certs/counter.crt --key=certs/counter.key
kubectl create secret tls counter-tls -n counter-b \
  --cert=certs/counter.crt --key=certs/counter.key

helm install counter-a helm/counter -n counter-a \
  --set ingress.host=counter-a.localtest.me
helm install counter-b helm/counter -n counter-b \
  --set ingress.host=counter-b.localtest.me

helm test counter-a -n counter-a
helm test counter-b -n counter-b
```

Проверка двух адресов:

```bash
curl --cacert certs/counter.crt https://counter-a.localtest.me/api/counter
curl --cacert certs/counter.crt https://counter-b.localtest.me/api/counter
```

Обновление порта меняет checksum конфигурации, поэтому Pod перезапускается:

```bash
helm upgrade counter-a helm/counter -n counter-a \
  --set ingress.host=counter-a.localtest.me \
  --set backend.port=8082

kubectl rollout status deployment/counter-a-backend -n counter-a
helm history counter-a -n counter-a
helm rollback counter-a 1 -n counter-a
helm history counter-a -n counter-a
```

После `helm install` автоматически запускается post-install проверка backend.

## Скриншот

Страница открыта по адресу `https://counter.localtest.me`, кнопка увеличивает
счётчик.

![Работающее приложение counter](screenshots/Screenshot%202026-10-25%20at%2020.20.53.png)

## Логи выполнения

Ниже приведены фактические результаты запуска на Ubuntu 22.04 ARM64.

### Кластер и образы

```bash
kubectl get nodes -o wide
```

```text
NAME                   STATUS   ROLES           VERSION   INTERNAL-IP   OS-IMAGE                       KERNEL-VERSION               CONTAINER-RUNTIME
course-control-plane   Ready    control-plane   v1.37.0   172.21.0.2   Debian GNU/Linux 13 (trixie)   5.15.0-198-generic (arm64)   containerd://2.3.4
course-worker          Ready    <none>          v1.37.0   172.21.0.4   Debian GNU/Linux 13 (trixie)   5.15.0-198-generic (arm64)   containerd://2.3.4
course-worker2         Ready    <none>          v1.37.0   172.21.0.3   Debian GNU/Linux 13 (trixie)   5.15.0-198-generic (arm64)   containerd://2.3.4
```

```bash
kubectl get pods -n ingress-nginx
docker images | grep counter
```

```text
NAME                                        READY   STATUS      RESTARTS
ingress-nginx-admission-create-79mw2        0/1     Completed   0
ingress-nginx-admission-patch-5q92t         0/1     Completed   0
ingress-nginx-controller-6d97975bd5-7qjsb   1/1     Running     0

counter-backend:local    bea6debcaceb   260MB   57.5MB
counter-frontend:local   1ab40230a905   440MB   104MB
```

### Установленные ресурсы

```bash
kubectl get all -n counter
kubectl get ingress,pvc -n counter
```

```text
NAME                                  READY   STATUS    RESTARTS
pod/backend-5b7cb5d4fc-48f4h         1/1     Running   0
pod/frontend-db8966d47-qfq8v          1/1     Running   0
pod/node-exporter-lgqt2               1/1     Running   0
pod/node-exporter-qybzg               1/1     Running   0
pod/node-exporter-xn46d               1/1     Running   0
pod/postgres-0                        1/1     Running   0

NAME                 TYPE        CLUSTER-IP     PORT(S)
service/backend      ClusterIP   10.96.156.2   8081/TCP
service/frontend     ClusterIP   10.96.107.39  8080/TCP
service/postgres     ClusterIP   None           5432/TCP

NAME                                     DESIRED   CURRENT   READY
daemonset.apps/node-exporter             3         3         3
deployment.apps/backend                  1/1       1         1
deployment.apps/frontend                 1/1       1         1
statefulset.apps/postgres                1/1

NAME                                  CLASS   HOSTS                  ADDRESS     PORTS
ingress.networking.k8s.io/counter     nginx   counter.localtest.me   localhost   80, 443

NAME                                      STATUS   CAPACITY   ACCESS MODES   STORAGECLASS
persistentvolumeclaim/data-postgres-0     Bound    1Gi        RWO            standard
```

### HTTPS, Ingress и Services

```bash
curl -4 --cacert certs/counter.crt \
  https://counter.localtest.me/api/counter
curl -4 --cacert certs/counter.crt -X POST \
  https://counter.localtest.me/api/counter
```

```text
{"count": 3}
{"count": 4}
```

Проверка сертификата не отключалась: используется `--cacert`, флаг `-k`
отсутствует.

```bash
kubectl get ingress counter -n counter
kubectl get service backend frontend -n counter
```

```text
NAME      CLASS   HOSTS                  ADDRESS     PORTS
counter   nginx   counter.localtest.me   localhost   80, 443

NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
backend    ClusterIP   10.96.156.2    <none>        8081/TCP
frontend   ClusterIP   10.96.107.39   <none>        8080/TCP
```

Оба Service имеют тип `ClusterIP`. Внешний HTTPS-трафик принимает Ingress:
`/api` направляется в backend, `/` — во frontend.

### Конфигурация, ресурсы и QoS

```bash
kubectl get pod -n counter -l app=backend \
  -o jsonpath='{.items[0].spec.containers[0].ports[0].containerPort}{"\n"}'
kubectl get pod -n counter -l app=backend \
  -o jsonpath='{.items[0].status.qosClass}{"\n"}'
kubectl get deployment backend -n counter \
  -o jsonpath='{.spec.template.spec.containers[0].resources}{"\n"}'
```

```text
8081
Guaranteed
{"limits":{"cpu":"100m","memory":"128Mi"},"requests":{"cpu":"100m","memory":"128Mi"}}
```

### Readiness и отсутствие простоя

После установки заведомо неправильного пути readiness одновременно видны
старый готовый и новый неготовый Pod:

```text
NAME                       READY   STATUS
backend-58b5b7f8b9-gckn7   0/1     Running
backend-5b7cb5d4fc-48f4h   1/1     Running
```

```bash
kubectl get endpointslice -n counter \
  -l kubernetes.io/service-name=backend \
  -o jsonpath='{range .items[*].endpoints[*]}{.addresses[0]}{" ready="}{.conditions.ready}{"\n"}{end}'
```

```text
10.244.1.3 ready=true
10.244.2.7 ready=false
```

```bash
for i in 1 2 3; do
  curl -4 --cacert certs/counter.crt \
    https://counter.localtest.me/api/counter
  echo
done
```

```text
{"count": 3}
{"count": 3}
{"count": 3}
```

Неготовый адрес остаётся в EndpointSlice с `ready=false`, но Service не
отправляет на него трафик. Откат завершился успешно:

```text
deployment.apps/backend rolled back
deployment "backend" successfully rolled out
```

### Масштабирование backend

```bash
kubectl scale deployment/backend -n counter --replicas=3
kubectl rollout status deployment/backend -n counter
kubectl get pods -n counter -l app=backend -o wide
```

```text
deployment.apps/backend scaled
deployment "backend" successfully rolled out
NAME                       READY   STATUS    IP           NODE
backend-5b7cb5d4fc-48f4h   1/1     Running   10.244.1.3   course-worker2
backend-5b7cb5d4fc-4tg8f   1/1     Running   10.244.2.5   course-worker
backend-5b7cb5d4fc-kd4qh   1/1     Running   10.244.2.6   course-worker
```

```text
{"count": 1}
{"count": 1}
{"count": 1}
{"count": 2}
{"count": 2}
{"count": 3}
```

Значение прыгает, потому что каждый Pod хранит отдельный счётчик в памяти.

### Неудачное обновление и восстановление

```bash
kubectl set image deployment/backend -n counter \
  backend=counter-backend:not-found
kubectl rollout status deployment/backend -n counter --timeout=30s
```

```text
Waiting for deployment "backend" rollout to finish: 1 old replicas are pending termination...
error: timed out waiting for the condition
```

Старый Pod остался готовым. После восстановления исходного манифеста:

```bash
kubectl apply -n counter -f k8s/app.yaml
kubectl rollout status deployment/backend -n counter
curl -4 --cacert certs/counter.crt \
  https://counter.localtest.me/api/counter
```

```text
deployment.apps/backend configured
deployment "backend" successfully rolled out
{"count": 3}
```

### PostgreSQL и постоянные данные

Запись в таблице перед пересозданием:

```text
 id | value
----+-------
  1 |    10
(1 row)
```

После удаления StatefulSet PVC сохранился:

```bash
kubectl delete statefulset postgres -n counter
kubectl get pvc -n counter
```

```text
statefulset.apps "postgres" deleted from counter namespace
NAME              STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
data-postgres-0   Bound    pvc-be69a65b-e771-4600-8489-1d0320390cc6   1Gi        RWO            standard
```

После повторного применения `k8s/postgres.yaml`:

```text
configmap/postgres-init unchanged
service/postgres unchanged
statefulset.apps/postgres created
pod/postgres-0 condition met

 id | value
----+-------
  1 |    10
(1 row)
```

Это подтверждает стабильное имя `postgres-0.postgres`, автоматическое
создание таблицы и сохранение данных в PVC.

### Node Exporter, RBAC и CronJob

```bash
kubectl get daemonset,pod -n counter -l app=node-exporter -o wide
```

```text
NAME                             DESIRED   CURRENT   READY
daemonset.apps/node-exporter     3         3         3
```

Node Exporter запущен на всех трёх нодах, включая control-plane.

```bash
kubectl auth can-i update deployment/frontend -n counter \
  --subresource=scale \
  --as=system:serviceaccount:counter:frontend-scaler
kubectl auth can-i delete pods -n counter \
  --as=system:serviceaccount:counter:frontend-scaler
```

```text
yes
no
```

ServiceAccount дополнительно имеет только необходимое право `get` на
Deployment `frontend`: оно требуется команде `kubectl scale`. Изменять другие
Deployment или удалять Pod он не может.

```bash
kubectl create job "$JOB_UP" -n counter \
  --from=cronjob/frontend-scale-up
kubectl wait -n counter --for=condition=complete \
  job/"$JOB_UP" --timeout=120s
kubectl logs job/"$JOB_UP" -n counter
kubectl get deployment frontend -n counter
```

```text
job.batch/scale-up-1792948209 created
job.batch/scale-up-1792948209 condition met
deployment.apps/frontend scaled
NAME       READY   UP-TO-DATE   AVAILABLE
frontend   3/3     3            3
```

CronJob используют `Europe/Moscow`: scale-up запускается в 09:00, scale-down
— в 18:00.

### Helm: две установки, тест, upgrade и rollback

```bash
helm test counter-a -n counter-a
helm test counter-b -n counter-b
```

```text
NAME: counter-a
STATUS: deployed
REVISION: 1
TEST SUITE: counter-a-test
Phase: Succeeded

NAME: counter-b
STATUS: deployed
REVISION: 1
TEST SUITE: counter-b-test
Phase: Succeeded
```

Обе установки имеют отдельные Deployment, Service и Ingress:

```text
counter-a-backend    1/1   Running
counter-a-frontend   1/1   Running
counter-a.localtest.me

counter-b-backend    1/1   Running
counter-b-frontend   1/1   Running
counter-b.localtest.me
```

```bash
curl -4 --cacert certs/counter.crt \
  https://counter-a.localtest.me/api/counter
curl -4 --cacert certs/counter.crt \
  https://counter-b.localtest.me/api/counter
```

```text
{"count": 0}
{"count": 0}
```

Изменение `backend.port` пересоздало Pod:

```text
old=counter-a-backend-845897566d-rt79m
new=counter-a-backend-7c46f4756d-xbqj8
```

```bash
helm history counter-a -n counter-a
```

```text
REVISION  STATUS      DESCRIPTION
1         superseded  Install complete
2         superseded  Upgrade complete
3         deployed    Rollback to 1
```

После rollback приложение продолжает работать:

```text
{"count": 0}
```
