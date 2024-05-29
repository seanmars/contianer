# Observability Service

可觀測性服務 docker compose，包含以下服務：

- [Jaeger](https://www.jaegertracing.io/) - Jaeger: open source, distributed tracing platform
- [Prometheus](https://prometheus.io/) - Power your metrics and alerting with the leading open-source monitoring solution.
- [OpenSearch](https://opensearch.org/) - OpenSearch is the flexible, scalable, open-source way to build solutions for data-intensive applications.
- [Grafana](https://grafana.com/) - Query, visualize, alert on, and understand your data no matter where it's stored. With Grafana you can create, explore, and share all of your data through beautiful, flexible dashboards.

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

## Prometheus

- Access the Prometheus dashboard at [http://localhost:9090](http://localhost:9090).
- Update targets on runtime
  
  ```
  POST http://localhost:9090/-/reload
  ```