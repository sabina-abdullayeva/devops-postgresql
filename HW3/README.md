## 1. Мониторинг

Запуск:

```bash
cd monitoring
docker compose up -d
docker compose ps
curl http://localhost:9100/metrics
```

Адреса:

- Prometheus: http://localhost:9090
- Targets: http://localhost:9090/targets
- Grafana: http://localhost:3000
- логин и пароль Grafana: `admin` / `admin`

Источник данных Prometheus и дашборд подключаются автоматически через
provisioning. Дашборд `PainOps` находится в папке `Konoha`.

На дашборде есть переменная `instance` и четыре панели:

- загрузка CPU;
- использование RAM;
- свободное место на диске;
- входящий и исходящий сетевой трафик.

Нода имеет понятную метку `instance=konoha-vm1`. Тематические названия
не меняют обязательные Prometheus-метрики и фильтр
`{instance=~"$instance"}`.

## 2. Docker-образы

Сборка:

```bash
cd counter-backend

docker build -f Dockerfile.single -t pain-counter:single .
docker build -f Dockerfile.multistage -t pain-counter:multistage .
docker build -f Dockerfile.nonroot -t pain-counter:nonroot .
```

Проверка первого образа:

```bash
docker run -d --name pain-counter-single -p 8081:8080 pain-counter:single
curl http://localhost:8081/api/counter
curl -X POST http://localhost:8081/api/counter
docker exec pain-counter-single id
docker rm -f pain-counter-single
```

Проверка второго образа:

```bash
docker run -d --name pain-counter-multistage -p 8082:8080 pain-counter:multistage
curl http://localhost:8082/api/counter
curl -X POST http://localhost:8082/api/counter
docker exec pain-counter-multistage id
docker rm -f pain-counter-multistage
```

Проверка третьего образа:

```bash
docker run -d --name pain-counter-nonroot -p 8083:8080 pain-counter:nonroot
curl http://localhost:8083/api/counter
curl -X POST http://localhost:8083/api/counter
docker exec pain-counter-nonroot id
docker rm -f pain-counter-nonroot
```

Ожидаемые ответы API:

```text
{"count": 0}
{"count": 1}
```

В образах `single` и `multistage` команда `id` показывает `uid=0(root)`.
В образе `nonroot` Пейн лишается привилегий root: команда показывает
`uid=1001(nagato)`.

Размеры можно посмотреть командой:

```bash
docker image ls pain-counter
```

| Образ | Размер |
| --- | --- |
| `pain-counter:single` | 390 MB |
| `pain-counter:multistage` | 260 MB |
| `pain-counter:nonroot` | 261 MB |

Фактические результаты проверок находятся в [`logs/docker.txt`](logs/docker.txt),
а состояние мониторинга и метрики — в [`logs/mon.txt`](logs/mon.txt).

## Скриншоты для сдачи

После запуска нужно положить в папку `screenshots` три скриншота:

1. страница Prometheus Targets, где цель `node` имеет статус `UP`;
2. успешная проверка источника данных Prometheus в Grafana;
3. дашборд со всеми четырьмя панелями.
