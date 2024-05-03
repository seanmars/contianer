## Prometheus & Grafana

Project structure:
```
.
├── compose.yaml
├── grafana
│   ├── dashboards
│   │   ├── aspnetcore-endpoint.json
│   │   └── aspnetcore.json
│   └── datasource.yml
├── prometheus
│   └── prometheus.yml
└── README.md
```

## Deploy with docker compose

- Start the containers in the background.

```shell
docker-compose up -d
```

- Stop and remove the containers. Use `-v` to remove the volumes.

```shell
$ docker compose down -v
```