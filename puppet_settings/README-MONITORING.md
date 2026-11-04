Monitori*g Stack Deployment
Purpose
De*loy centralized monitoring using: ```text node_exporter ↓ VictriaMetrics ↓ Grafana


--*

# Infrastructure

```text
node_e*porter
      ↓
VictoriaMetrics
   * *↓
Grafana

Vector
     *↓
ClickHouse
*``

---

# Node Exporter

## Deplo*ment

Node Exporter is deployed vi* Puppet profile:

```text
profile:*node_exporter
Installed on:

``text clickhouse.train.local.a1.b git2.train.local.a1.by haproxy.trin.local.a1.by mon.train.local.a1.y node2-01.train.local.a1.by node202.train.local.a1.by puppet2.trainlocal.a1.by vault.train.local.a1.b*


---

## Verification

```bas*
curl http://127.0.0.1:9100/metrics
Expected:

# HELP
# *YPE
VictoriaMetrics
#* Binary

/usr/local/bin/vi*toria-metrics*prod
Data Directory
text /var/lib/victoriametrics

Configuration
/*tc/victoriametrics/prometheus.yml
*``

```yaml
global:
  scrape_inter*al: 15s

scrape_configs:

  - job_*ame: node_exporter

    static_con*igs:
      - targets:
          - *lickhouse.train.local.a1.by:9100
 *        - git2.train.local.a1.by:9*00
          - haproxy.train.local*a1.by:9100
         *- mon.train.local.a1.by:9100
     *    - node2-01.train.local.a1.by:9*00
          - node2-02.train.loca*.a1.by:9100
          - puppet2.tr*in.local.a1.by:9100
          - va*lt.train.local.a1.by:9100
---*

Systemd Service
[Unit]*Description=VictoriaMetrics

[Service]
ExecStart=/usr/local/bin/victo*ia-metrics-prod \
  -storageDataPa*h=/var/lib/victoriametrics \
  -ht*pListenAddr=:8428 \
  -promscrape.*onfig=/etc/victoriametrics/prometh*us.yml

Restart=always

[Install]
*antedBy=multi-user.target*```

---

## Verification

Targets*

```bash
curl http://127.0.0.1:8428/api/v1/targets
Expected:

*json "health":"up"


for all h*sts.

---

## Query Example

```ba*h
curl \
'http://127.0.0.1:8428/ap*/v1/query?query=up'
Expected:*

value = 1
for all mo*itored servers.

Grafana
#* Installation

Package:

g*afana-12.4.11-3.1.x86_64.rpm
*--

Service
sudo syste*ctl enable grafana-server
sudo sys*emctl start grafana-server
--*

Verification*
curl -I http://127.0.0.1:3000
Expected:

HTTP/1*1 302 Found
Location: /login
*--

Grafana Data Source
Crea*e Data Source
Type:

Prom*theus
URL:

http://12*.0.0.1:8428
Save:

Sa*e & Test
Expected:

S*ccessfully queried the Prometheus *PI
Explore Metrics
##*Host Availability

up
``*

Expected:

```text
clickhouse
gi*2
haproxy
mon
node2-01
node2-02
pu*pet2
vault
with value:

1
CPU Usage
Memory Usage
```promql (node_memory_MemTotal_byes - node_memory_MemAvailable_byte*) / node_memory_MemTotal_bytes * 1*0


---

## Load Average

```pr*mql
node_load1
Disk U*age
100 -
(
  node_file*ystem_avail_bytes
  /
  node_files*stem_size_bytes
) * 100
Dashboard Import
Node Exporte* Full
Import dashboard:

*D: 1860
Select*datasource:

VictoriaMetri*s
External Access
*rafana
Internal:

http://*92.168.225.4:3000
External NA*:

178.163.233.114:3000
→ *92.168.225.4:3000
Required:

DNAT rule
TCP/3000 allowed
Gr*fana listening on 0.0.0.0:3000
--*

Final Architecture
Gi*2
   |
   v
Puppet2
   |
   +------ Vault
   |
   +------ Managed Servers

Node Exporter
       |
       v
VictoriaMetrics
       |
       v
Grafana

Vector
       |
       v
ClickHouse
