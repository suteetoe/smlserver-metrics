# Agent Project Overview

เอกสารนี้ใช้เป็นบริบทหลักให้ agent ที่เข้ามาช่วยแก้ไขหรือ deploy โปรเจกต์ `smlserver-metrics`

## เป้าหมายของโปรเจกต์

โปรเจกต์นี้สร้างระบบ Monitoring สำหรับ SML ERP ที่จะติดตั้งและรันบน server IP `192.168.2.213`

เป้าหมายคือเฝ้าระวังการทำงานของ server และ application ที่เกี่ยวข้อง เพื่อลดความเสี่ยงจากการหยุดทำงานของ host, web service, database และ reverse proxy ที่อยู่ใน stack นี้

ระบบที่ต้องติดตามหลัก ๆ คือ:

- Host metrics: CPU, Memory, Disk, Network
- Docker/container metrics
- SML WebService metrics ผ่าน JMX Exporter
- PostgreSQL metrics ผ่าน Postgres Exporter
- Traefik traffic metrics
- Docker logs ของ service สำคัญ
- Dashboard และ alert สำหรับดูสถานะรวมของ SML ERP

## Server เป้าหมาย

- Target server: `192.168.2.213`
- Stack ทั้งหมดรันด้วย Docker Compose
- Docker network ที่ใช้ร่วมกัน: `sml_service_network`
- หลังจากแก้ไข config ใน repo นี้ ต้องส่งไฟล์ที่เกี่ยวข้องไปรันบน server `192.168.2.213`

## โครงสร้างโปรเจกต์

```text
.
├── README.md
├── AGENTS.md
└── src/
    ├── monitoring_host/
    │   ├── docker-compose.yaml
    │   ├── .env
    │   ├── conf/
    │   │   └── loki-config.yml
    │   ├── dashboard/
    │   │   ├── sml-server-overview.json
    │   │   ├── Traefik.json
    │   │   ├── JMX Dashboard(Basic).json
    │   │   └── PostgreSQL Database.json
    │   └── grafana/provisioning/
    │       ├── dashboards/dashboards.yml
    │       └── datasources/prometheus.yml
    └── service_host/
        ├── docker-compose.yaml
        ├── .env
        └── conf/
            ├── config.alloy
            └── jmx-config.yaml
```

## บทบาทของแต่ละส่วน

### `src/monitoring_host`

เป็นฝั่งรับและแสดงผล monitoring data

- `prometheus`: รับ metrics ผ่าน remote write และให้ Grafana query
- `loki`: รับ logs จาก Alloy
- `grafana`: แสดง dashboard ผ่าน path `/grafana`
- `dashboard/`: เก็บ dashboard JSON ที่ Grafana provision เข้าไปใช้งาน
- `grafana/provisioning/`: เก็บ datasource และ dashboard provisioning

### `src/service_host`

เป็นฝั่ง application และ collector

- `traefik`: reverse proxy และ traffic metrics endpoint ที่ port `9091`
- `postgresql`: database ของ SML ERP
- `postgres_exporter`: export PostgreSQL metrics ที่ port `9187`
- `smljavawebservice`: SML WebService บน Tomcat และเปิด JMX metrics ที่ port `8081`
- `node_exporter`: export host metrics ที่ port `9100`
- `cadvisor`: export container metrics
- `alloy`: scrape metrics และ collect Docker logs แล้วส่งไป Prometheus/Loki

## Data Flow

```text
service_host containers
  -> Alloy scrape metrics
  -> Prometheus remote write
  -> Grafana dashboard

Docker logs
  -> Alloy loki.source.docker
  -> Loki
  -> Grafana Explore / dashboard
```

Metrics endpoints สำคัญ:

- SML WebService JMX: `http://192.168.2.213:8081/metrics`
- PostgreSQL exporter: `http://192.168.2.213:9187/metrics`
- Traefik metrics: `http://192.168.2.213:9091/metrics`
- Prometheus: `http://192.168.2.213:9090`
- Grafana: `http://192.168.2.213/grafana`

## แนวทางการทำงานสำหรับ Agent

ก่อนแก้ไข config ให้ตรวจสอบไฟล์ที่เกี่ยวข้องก่อนเสมอ:

- แก้ metrics/log collection: ดู `src/service_host/conf/config.alloy`
- แก้ JMX metrics mapping: ดู `src/service_host/conf/jmx-config.yaml`
- แก้ service/container: ดู `src/service_host/docker-compose.yaml`
- แก้ Prometheus/Loki/Grafana: ดู `src/monitoring_host/docker-compose.yaml`
- แก้ datasource/dashboard provisioning: ดู `src/monitoring_host/grafana/provisioning/`
- แก้ dashboard: ดู `src/monitoring_host/dashboard/`

หลักการแก้ไข:

- รักษา Docker network `sml_service_network`
- หลีกเลี่ยงการเปลี่ยนชื่อ container โดยไม่จำเป็น เพราะ Alloy และ dashboard อ้างอิงชื่อเหล่านี้
- ถ้าเปลี่ยน endpoint, port, job label หรือ container name ต้องตรวจสอบ dashboard และ query ที่เกี่ยวข้องด้วย
- อย่า commit หรือเผยแพร่ค่า secret จาก `.env`
- ถ้าเพิ่ม exporter หรือ service ใหม่ ให้เพิ่ม scrape config ใน Alloy และ dashboard/alert ที่จำเป็น

## Workflow หลังแก้ไข Config

หลังจากแก้ไข config ให้ตรวจสอบ syntax และสถานะไฟล์ก่อนส่งขึ้น server:

```bash
git diff -- README.md AGENTS.md src
docker compose -f src/monitoring_host/docker-compose.yaml config
docker compose -f src/service_host/docker-compose.yaml config
```

จากนั้นส่งไฟล์ไปยัง server `192.168.2.213` ด้วย `scp` หรือ `rsync` ตาม path ที่ใช้งานจริงบน server

ตัวอย่างด้วย `rsync`:

```bash
rsync -avz \
  --exclude '.git' \
  --exclude 'monitoring_host/grafana-data' \
  --exclude 'monitoring_host/prometheus-data' \
  --exclude 'monitoring_host/loki-data' \
  --exclude 'service_host/postgresql' \
  --exclude 'service_host/tomcat_temp' \
  ./src/ root@192.168.2.213:/data/
```

คำสั่งนี้จะส่งทุก folder ที่อยู่ใน `src/` ไปไว้ใต้ `/data/` บน server เช่น `src/monitoring_host` จะไปเป็น `/data/monitoring_host` และ `src/service_host` จะไปเป็น `/data/service_host`

คำสั่งนี้ตั้งใจให้เครื่อง local เป็น source หลักสำหรับ config/source files แต่ไม่ใช้ `--delete` เพื่อไม่ลบไฟล์ runtime หรือ data directory ที่ container สร้างเพิ่มบน server

ตัวอย่าง deploy เฉพาะ config ที่แก้:

```bash
rsync -avz src/service_host/conf/config.alloy root@192.168.2.213:/data/service_host/conf/config.alloy
rsync -avz src/service_host/conf/jmx-config.yaml root@192.168.2.213:/data/service_host/conf/jmx-config.yaml
rsync -avz src/monitoring_host/conf/loki-config.yml root@192.168.2.213:/data/monitoring_host/conf/loki-config.yml
```

หลังส่งไฟล์แล้ว SSH เข้า server และ recreate service ที่เกี่ยวข้อง:

```bash
ssh root@192.168.2.213
cd /data
docker compose -f monitoring_host/docker-compose.yaml up -d
docker compose -f service_host/docker-compose.yaml up -d
```

ถ้าแก้เฉพาะ Alloy config:

```bash
docker compose -f service_host/docker-compose.yaml up -d --force-recreate alloy
```

ถ้าแก้ Grafana dashboard/provisioning:

```bash
docker compose -f monitoring_host/docker-compose.yaml up -d --force-recreate grafana
```

## Verification Checklist

หลัง deploy ให้ตรวจสอบ:

- `docker ps` ต้องเห็น container หลักทำงานครบ
- `docker logs alloy --tail=100` ไม่มี error เรื่อง scrape หรือ remote write
- `docker logs prometheus --tail=100` ไม่มี error สำคัญ
- `docker logs loki --tail=100` ไม่มี error สำคัญ
- เปิด `http://192.168.2.213:9090/targets` หรือ query metrics ใน Prometheus ได้
- เปิด Grafana ที่ `http://192.168.2.213/grafana`
- Dashboard มีข้อมูล Host, Docker, SML WebService, PostgreSQL และ Traefik
- Logs จาก container สำคัญค้นหาใน Loki/Grafana ได้

## Alert ที่ควรมี

ระบบนี้ควรมี alert อย่างน้อย:

- SML WebService down
- PostgreSQL down
- Traefik down
- Host memory usage มากกว่า 80%
- Disk usage สูงผิดปกติ
- Container restart บ่อยผิดปกติ
- Prometheus/Loki/Grafana/Alloy down

## หมายเหตุสำคัญ

- ไฟล์ `.env` มีค่าที่เกี่ยวข้องกับ runtime และอาจมี secret ให้ระวังก่อนแสดงผลหรือ commit
- `jmx_prometheus_javaagent-1.5.0.jar` ต้องอยู่ใน `service_host/conf/` บน server เพื่อให้ SML WebService export JMX metrics ได้
- ในเครื่อง local ไฟล์ source อยู่ใต้ `src/` แต่เมื่อ sync ด้วย `rsync ./src/ root@192.168.2.213:/data/` โครงสร้างบน server จะเป็น `/data/monitoring_host` และ `/data/service_host`
- `src/monitoring_host/docker-compose.yaml` และ `src/service_host/docker-compose.yaml` ใช้ network ภายนอกชื่อ `sml_service_network` ดังนั้นบน server ต้องมี network นี้อยู่ก่อน หรือสร้างด้วย:

```bash
docker network create sml_service_network
```
