# Setting Up a Caldera-Wildcat DAMN OT Lab with Zeek and ELK Using Docker Compose

## Links
[[caldera-ot]]
[[Mitre-wildcat-damn]]
[[zeek]]
[[OT-sec-concepts]]

As a cybersecurity expert specializing in lab environments, I've set up countless simulations like this to mimic real-world operational technology (OT) attacks. This guide walks you through creating a containerized lab where **Caldera** (from MITRE) acts as the attacker emulating Modbus/TCP threats against **Wildcat Dam** (MITRE's simulated OT dam control system using Modbus/TCP). We'll monitor traffic with **Zeek** (for detailed protocol logging) and analyze it in the **ELK Stack** (Elasticsearch for storage, Logstash/Filebeat for ingestion, Kibana for visualization and alerts).

This setup simulates a basic segmented network: an "OT network" for the attacker and target, with Zeek "sniffing" traffic in a monitoring role. It's designed for users with basic networking knowledge (e.g., understanding IPs and ports) but no deep OT expertise. We'll keep things straightforward, using Docker Compose for orchestration.

**Lab Overview**:
- **Attacker**: Caldera with Modbus plugin to send malicious Modbus commands (e.g., writing to coils to "open gates").
- **Target**: Wildcat Dam simulating a Modbus/TCP device (PLC-like) on port 5020.
- **Monitoring**: Zeek parses Modbus traffic into logs (e.g., reads/writes).
- **Analysis**: ELK ingests Zeek logs for searching, dashboards, and alerts (e.g., detect unauthorized writes).
- **Networks**: A shared "ot_net" for simulation traffic; ELK on a "mgmt_net" for safe access.

**Prerequisites**:
- Docker and Docker Compose installed (v20+; test with `docker --version`).
- A host machine with 8+ GB RAM, 4+ CPU cores, 50 GB disk (e.g., Ubuntu VM).
- Basic terminal skills; run commands with `sudo` if needed for privileges.
- Git for cloning repos.

**Potential Pitfalls**: Docker privilege issues (fix with `cap_add`), volume permissions (use `chmod 777` on log dirs), or plugin loading (restart containers). If traffic isn't captured, check Zeek's interface.

## 1. Docker Compose Setup

We'll create a `docker-compose.yml` file to spin up all components. First, prepare your project directory:

```
mkdir caldera-ot-lab && cd caldera-ot-lab
mkdir logs data {caldera-plugins, wildcat-dam}
```

### Custom Dockerfiles
We need custom images for Caldera (with Modbus plugin) and Wildcat Dam (Python-based).

- **For Caldera (with Modbus Plugin)**: Create `Dockerfile.caldera`:
  ```
  FROM ghcr.io/mitre/caldera:latest
  RUN apt-get update && apt-get install -y git
  RUN git clone https://github.com/mitre/modbus.git /opt/caldera/plugins/modbus
  RUN echo "- modbus" >> /opt/caldera/conf/default.yml  # Enable plugin
  EXPOSE 8888
  CMD ["python3", "server.py"]
  ```

- **For Wildcat Dam**: Clone the repo and create `Dockerfile.wildcat` in `wildcat-dam/`:
  ```
  cd wildcat-dam
  git clone https://github.com/mitre/wildcatdam.git .
  ```
  Then `Dockerfile.wildcat`:
  ```
  FROM python:3.12-slim
  WORKDIR /app
  COPY . .
  RUN pip install -r requirements.txt
  EXPOSE 5020
  CMD ["python", "dam_system.py"]
  ```

### Docker Compose File
Create `docker-compose.yml`:
```yaml
version: '3.8'
services:
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.14.3
    container_name: so-elasticsearch  # Reuse for familiarity
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false  # Lab only; enable for prod
      - ES_JAVA_OPTS=-Xms1g -Xmx1g
    networks:
      - mgmt_net
    volumes:
      - ./data/es:/usr/share/elasticsearch/data
    ports:
      - "9200:9200"

  kibana:
    image: docker.elastic.co/kibana/kibana:8.14.3
    container_name: so-kibana
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
    networks:
      - mgmt_net
    ports:
      - "5601:5601"
    depends_on:
      - elasticsearch

  filebeat:
    image: docker.elastic.co/beats/filebeat:8.14.3
    container_name: filebeat
    user: root  # For log access
    volumes:
      - ./logs:/logs:ro  # Mount Zeek logs
      - ./filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
    networks:
      - mgmt_net
    depends_on:
      - elasticsearch
    command: filebeat -e -strict.perms=false

  zeek:
    image: zeek/zeek:latest
    container_name: so-zeek
    network_mode: host  # To capture host traffic (assumes Docker host routes OT traffic)
    cap_add:
      - NET_ADMIN
      - NET_RAW
    volumes:
      - ./logs:/opt/zeek/logs  # Persist logs
    environment:
      - ZEEK_INTERFACE=any  # Capture all; adjust to eth0 if needed
    command: >
      bash -c "
        zeekctl install &&
        echo '@load base/protocols/modbus' >> /opt/zeek/share/zeek/site/local.zeek &&
        zeekctl deploy &&
        zeekctl start
      "
    depends_on:
      - wildcat-dam  # Start after target

  caldera:
    build:
      context: .
      dockerfile: Dockerfile.caldera
    container_name: caldera-attacker
    ports:
      - "8888:8888"  # Web UI
    networks:
      - ot_net
    volumes:
      - ./caldera-plugins:/opt/caldera/plugins  # For custom facts
    depends_on:
      - wildcat-dam

  wildcat-dam:
    build:
      context: ./wildcat-dam
      dockerfile: Dockerfile.wildcat
    container_name: wildcat-target
    networks:
      - ot_net
    ports:
      - "5020:5020"  # Modbus/TCP

networks:
  ot_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16  # Isolated OT sim
  mgmt_net:
    driver: bridge
    ipam:
      config:
        - subnet: 172.30.0.0/16  # For ELK access
```

Build and start: 
```
docker compose build
docker compose up -d
```
Verify: `docker compose ps` (all "Up"). Access Caldera at `http://localhost:8888` (default admin/SecretPassword; change it).

**Troubleshooting**: If build fails, check Git clones. For Zeek not capturing, ensure host firewall allows (e.g., `sudo ufw allow in on eth0`).

## 2. Configuration Steps: Simulating Modbus/TCP Attacks with Caldera

Caldera emulates attacks using "abilities" (TTPs). We'll configure it to target Wildcat Dam.

1. **Prepare Facts for Targeting**: Copy Wildcat facts to Caldera plugins (from Wildcat repo):
   ```
   cd wildcat-dam/caldera
   cp wildcat_dam_facts.yml ../../caldera-plugins/sources/
   ```
   Restart Caldera: `docker compose restart caldera`.

2. **Access Caldera UI**: Go to `http://localhost:8888`. Log in, go to **Adversaries** > Create Operation.
   - Name: "Modbus Flood".
   - Fact Source: "Wildcat Dam Facts" (select from dropdown).
   - Select adversary: "Generic OT Attacker" or default.
   - Add abilities: Search for "modbus" (e.g., "Write Multiple Coils" to simulate gate manipulation).
   - Set facts: `modbus.server.ip: 172.20.0.3` (Wildcat's IP; find with `docker inspect wildcat-target`), `modbus.server.port: 5020`.
   - Run the operation: Click **Run** to simulate attacks (e.g., repeated writes causing "flood").

3. **Simulate Traffic**: Operations will generate Modbus/TCP packets over `ot_net`. Monitor Wildcat console: `docker logs wildcat-target` for responses.

**Pitfall**: If no facts show, remount volume and restart. Test connectivity: From Caldera container, `docker exec -it caldera ping 172.20.0.3`.

## 3. Integrating Zeek: Monitoring Modbus Traffic

Zeek passively analyzes traffic, logging Modbus specifics like function codes (e.g., read/write requests).

1. **Enable Modbus Parsing**: Already in Compose (via `@load base/protocols/modbus`). This logs to `modbus.log` (requests/responses), `modbus/conversation.log` (sessions).

2. **Capture Traffic**: With `network_mode: host`, Zeek sees all host traffic. Ensure Docker routes OT traffic through the host interface (e.g., via `docker network connect` if needed). Generate attacks from Caldera to trigger logs.

3. **View Logs**: `tail -f logs/current/modbus.log`. Look for entries like:
   ```
   #fields	ts	uid	id.orig_h	id.orig_p	id.resp_h	id.resp_p	modbus.req_func	modbus.req_data	modbus.resp_func	modbus.resp_data
   1725168000.123456	C12345	172.20.0.2	12345	172.20.0.3	5020	16	\x01\x02...	16	\x00\x00...
   ```
   (Function 16 = Write Multiple Registers; anomalous data indicates attack.)

**Troubleshooting**: No logs? Run `zeekctl status` inside container (`docker exec -it so-zeek zeekctl status`). Add `@load policy/protocols/modbus/known-masters-slaves` to `local.zeek` for better tracking; redeploy.

For advanced: Install CISA ICSNPP-Modbus package via zkg in Zeek container, but skip for basics.

## 4. Setting Up ELK: Indexing Zeek Logs

ELK centralizes logs for analysis.

1. **Configure Filebeat**: Create `filebeat.yml`:
   ```
   filebeat.inputs:
   - type: log
     enabled: true
     paths:
       - /logs/current/modbus*.log
     fields:
       log_type: zeek_modbus
     fields_under_root: true

   output.elasticsearch:
     hosts: ["elasticsearch:9200"]
     index: "zeek-modbus-%{+yyyy.MM.dd}"
   ```
   This ships Modbus logs to Elasticsearch.

2. **Start Ingestion**: Filebeat auto-starts. Verify indices: `curl http://localhost:9200/_cat/indices` (see `zeek-modbus-*`).

3. **Access Kibana**: `http://localhost:5601`. Go to **Stack Management** > **Index Patterns** > Create "zeek-modbus-*". Select `@timestamp` as time field.

**Pitfall**: Permission errors? Run `chmod -R 777 logs`. If no data, check Filebeat logs: `docker logs filebeat`.

## 5. Creating Alerts and Visualizations in Kibana

Now, build dashboards and alerts for attack detection (e.g., spike in Modbus writes).

1. **Create Visualizations**:
   - Go to **Analytics** > **Dashboard** > **Create new**.
   - Add viz: **Lens** > From "zeek-modbus-*".
     - Bar chart: X-axis `@timestamp`, Y-axis "Count" filtered by `modbus.req_func: 16` (writes).
     - Pie chart: Split by `id.orig_h` (sources; highlight Caldera IP).
     - Table: Columns for `modbus.req_data` to inspect payloads.
   - Save as "Modbus OT Dashboard".

2. **Set Up Alerts**:
   - **Analytics** > **Alerts** > **Create rule** > **Elasticsearch query**.
     - Index: "zeek-modbus-*".
     - Query: `modbus.req_func: 16 AND cardinality(id.orig_h) > 1` (multiple sources writing—attack indicator).
     - Threshold: >5 events in 5min.
     - Actions: Email (configure connector) or webhook.
   - Name: "Modbus Write Flood Alert". Test by running Caldera operation.

3. **Test End-to-End**: Run Caldera attack > Check dashboard for spikes > Trigger alert.

**Pitfall**: Alerts not firing? Ensure time range covers events; refresh indices.

## Next Steps and Best Practices
- Scale: Add more networks with `macvlan` for realism.
- Teardown: `docker compose down -v`.
- Learn More: [Caldera Docs](https://caldera.readthedocs.io/), [Zeek Modbus](https://docs.zeek.org/en/master/scripts/base/protocols/modbus/), [ELK Guide](https://www.elastic.co/guide/en/elastic-stack/current/index.html).
- Safety: This is isolated; never expose to prod networks.

This lab gives you hands-on OT attack simulation—start with a simple "read" ability, then escalate. Questions? Ping me!