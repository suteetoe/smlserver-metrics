# SML Server Metrics


```mermaid
flowchart LR
  subgraph service_host["Service Host"]
    traefik["Traefik<br/>:80 / :8080 / :9091"]
    sml["SMLJavaWebService<br/>app :8080<br/>JMX metrics :8081"]
    postgres["PostgreSQL<br/>:5432"]

    postgres_exporter["Postgres Exporter<br/>:9187"]
    node_exporter["Node Exporter<br/>:9100"]
    cadvisor["cAdvisor<br/>:8080"]
    alloy_agent["Alloy Agent<br/>scrape metrics + collect logs"]
  end

  subgraph monitoring_host["Monitoring Host"]
    prometheus["Prometheus<br/>remote write receiver :9090"]
    loki["Loki<br/>log receiver :3100"]
    grafana["Grafana<br/>dashboard :3000"]
  end

  traefik --> sml
  postgres_exporter --> postgres

  alloy_agent -->|"scrape JMX metrics"| sml
  alloy_agent -->|"scrape PostgreSQL metrics"| postgres_exporter
  alloy_agent -->|"scrape host metrics"| node_exporter
  alloy_agent -->|"scrape container metrics"| cadvisor
  alloy_agent -->|"scrape Traefik metrics"| traefik
  alloy_agent -->|"collect Docker logs"| sml
  alloy_agent -->|"collect Docker logs"| postgres
  alloy_agent -->|"collect Docker logs"| traefik

  alloy_agent -->|"remote_write metrics<br/>http://monitoring-host:9090/api/v1/write"| prometheus
  alloy_agent -->|"push logs<br/>http://monitoring-host:3100/loki/api/v1/push"| loki

  grafana -->|"query metrics"| prometheus
  grafana -->|"query logs"| loki

```

## purpose 

[x] Setup monitoring host (Prometheus, Grafana, Loki)
[x] Add persistent storage for Prometheus metrics
[x] Add persistent storage for Loki logs
[x] Setup metrics+log collector (Alloy)
[x] Config alloy for collecting Host metrics (CPU, Memory, Disk, Network)
[x] Config alloy for collecting Web Service metrics (JMX)
[x] Config alloy for collecting Database metrics (PostgreSQL)
[x] Config alloy for collecting traffic metrics (Traefik)
[x] Create dashboards for monitoring smlerp system (Host, Web Service, Database, traffic)
[x] Logging pipeline (Alloy -> Loki)
[ ] Provision Loki datasource in Grafana
[ ] Add log panels to Grafana dashboard
[ ] Alert when web service is down
[ ] Alert when database is down
[ ] Alert when memory threshold more than 80 percent


## Installation

1. Download JMX Exporter from https://github.com/prometheus/jmx_exporter and place it in the `conf` folder.
2. Create data directory for grafana 
```
mkdir grafana-data
chown 472:472 grafana-data

mkdir loki-data
chown -R 10001:10001 loki-data

mkdir prometheus-data
chown -R 65534:65534 prometheus-data

```
3. Start the containers
```
docker-compose up -d
```
4. Access Grafana at http://host.to/grafana

## Metrics

JMX - http://192.168.2.213:8081/metrics
PostgreSQL - http://192.168.2.213:9187/metrics
Traefik - http://192.168.2.213:9091/metrics

Prometheus - http://192.168.2.213:9090/metrics

## Grafana

http://192.168.2.213/grafana

## Prometheus

http://192.168.2.213:9090



Force Recreate alloy containers
```
docker compose up -d --force-recreate alloy
```


## Autoheal

```
docker inspect --format='{{json .State.Health}}' sml_webservice
docker logs autoheal --tail=100
```
