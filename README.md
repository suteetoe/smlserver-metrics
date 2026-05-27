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
[x] Setup metrics+log collector (Alloy)
[ ] Config alloy for collecting Host metrics (CPU, Memory, Disk, Network)
[ ] Config alloy for collecting Web Service metrics (JMX)
[ ] Config alloy for collecting Database metrics (PostgreSQL)
[ ] Config alloy for collecting traffic metrics (Traefik)
[ ] Create a dashboard for monitoring smlerp system (Host, Web Service, Database, traffic)
[ ] Alert when web service is down
[ ] Alert when database is down
[ ] Alert when memory threshold more than 80 percent
[ ] Logging 


## Installation

1. Download JMX Exporter from https://github.com/prometheus/jmx_exporter and place it in the `conf` folder.
2. Create data directory for grafana 
```
mkdir grafana-data
chown 472:472 grafana-data
```
3. Start the containers
```
docker-compose up -d
```
4. Access Grafana at http://localhost:3000
5. Add Prometheus as a data source
6. Import the dashboard from https:/na.com/grafana/dashboards/11376/grafa

## Metrics

JMX - http://192.168.2.213:8081/metrics
PostgreSQL - http://192.168.2.213:9187/metrics
Prometheus - http://192.168.2.213:9090/metrics

## Grafana

http://192.168.2.213/grafana

## Prometheus

http://192.168.2.213:9090



Force Recreate alloy containers
```
docker compose up -d --force-recreate alloy
```