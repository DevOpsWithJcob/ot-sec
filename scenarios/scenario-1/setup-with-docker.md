# Setting Up the Cybersecurity Lab Environment Using Docker Compose

## Links:
[[scenario1-setup]]
[[tools-list]]


This guide adapts the previous lab setup to use Docker Compose for easier deployment and management. All components (Caldera with OT plugin, Wildcat Dam, Zeek, and ELK Stack with Filebeat for log shipping) will run as Docker services. To simplify network traffic monitoring, we'll use `network_mode: host` for all services, allowing communication via localhost (127.0.0.1) and enabling Zeek to monitor the loopback interface (`lo`) for inter-service traffic. This approach is suitable for a lab but not recommended for production due to reduced isolation.

Assumptions:
- Docker and Docker Compose v2+ installed on Ubuntu 22.04 LTS (or similar).
- At least 8GB RAM, 4 CPUs, 50GB disk.
- Run everything on a single host/VM for simplicity.
- Latest versions as of October 2025: Caldera v5.1.0, Wildcat Dam (latest from repo), Zeek 7.0, Elastic Stack 9.x.

**Potential Pitfalls:**
- Host network mode exposes ports directly on the host—ensure no conflicts (e.g., stop local services on ports 8888, 502, 9200, 5601).
- On non-Linux Docker (e.g., Mac/Windows), host network may not fully support interface monitoring; use Linux for best results.
- High resource usage; monitor with `docker stats`.
- If traffic isn't captured on `lo`, verify with `tcpdump -i lo port 502` on the host during tests.

## Prerequisites
1. Install Docker and Docker Compose:
   ```
   sudo apt update && sudo apt install -y docker.io docker-compose
   sudo systemctl start docker && sudo systemctl enable docker
   ```
2. Create a project directory:
   ```
   mkdir lab-env && cd lab-env
   ```
3. Clone repositories:
   ```
   git clone https://github.com/mitre/caldera.git --recursive
   cd caldera
   git submodule add https://github.com/mitre/caldera-ot plugins/caldera-ot
   cd ..
   git clone https://github.com/mitre/wildcatdam.git
   ```

## Step 1: Prepare Custom Dockerfiles
### 1.1: Caldera Dockerfile (with OT Plugin)
Use Caldera's official Dockerfile but ensure the OT plugin is included via the submodule. No changes needed if cloned as above—the build will include it.

### 1.2: Wildcat Dam Dockerfile
Create `wildcatdam/Dockerfile`:
```
FROM python:3.12-slim

WORKDIR /app

COPY . /app

RUN pip install --no-cache-dir -r requirements.txt

CMD ["python", "wildcat_dam.py"]
```

## Step 2: Create Configuration Files
### 2.1: Caldera Configuration (Enable OT Plugin)
Create `caldera/conf/local.yml`:
```
plugins:
  enabled:
    - stockpile
    - training
    - caldera-ot  # Enables Modbus abilities
```

### 2.2: Wildcat Dam Configuration
Create or edit `wildcatdam/config/wildcat_dam.yaml`:
```
host: 0.0.0.0
port: 502
slave_id: 1
registers:
  - address: 0
    value: 100
  - address: 40001
    value: false
```

### 2.3: Zeek Configuration
Create `zeek/local.zeek`:
```
@load protocols/modbus
redef exit_only_after_terminate = F;
```

Create `zeek/networks.cfg` (define lab network):
```
127.0.0.0/8    Loopback Network
```

### 2.4: ELK Configurations
Create directories and files based on standard ELK setup.

- `elasticsearch/config/elasticsearch.yml`:
  ```
  network.host: 0.0.0.0
  http.port: 9200
  discovery.type: single-node
  ```

- `logstash/config/logstash.yml`:
  ```
  http.host: "0.0.0.0"
  ```

- `logstash/pipeline/logstash.conf` (placeholder; not strictly needed for Filebeat direct to ES):
  ```
  input { beats { port => 5044 } }
  output { elasticsearch { hosts => ["http://127.0.0.1:9200"] } }
  ```

- `kibana/config/kibana.yml`:
  ```
  server.host: "0.0.0.0"
  server.port: 5601
  elasticsearch.hosts: ["http://127.0.0.1:9200"]
  ```

- `filebeat/filebeat.yml`:
  ```
  filebeat.modules:
  - module: zeek
    conn:
      enabled: true
      var.paths: ["/zeek_logs/conn.log"]
    modbus:
      enabled: true
      var.paths: ["/zeek_logs/modbus*.log"]

  output.elasticsearch:
    hosts: ["http://127.0.0.1:9200"]
  ```

## Step 3: Create docker-compose.yml
In the project root (`lab-env`), create `docker-compose.yml`:
[[sc1-docker-compose]]
## Step 4: Build and Start the Environment
1. Build and start:
   ```
   docker compose up -d --build
   ```
   - This builds Caldera and Wildcat, pulls others, and starts all services.
2. Verify:
   - Caldera UI: `http://localhost:8888` (user: red, pass: admin).
   - Wildcat: Test with `telnet localhost 502` or modpoll.
   - Zeek: `docker logs zeek`—should show monitoring `lo`.
   - Kibana: `http://localhost:5601`.
3. Enable Zeek Module in Filebeat (one-time):
   ```
   docker exec -it filebeat filebeat modules enable zeek
   docker restart filebeat
   ```

## Step 5: Integration and Testing
### 5.1: Configure Components
- In Caldera UI, import OT abilities from `plugins/caldera-ot/abilities/`.
- Target Wildcat: Use `127.0.0.1:502` in facts/operations.
- Zeek captures on `lo`—generate traffic by running a Caldera Modbus operation.
- In Kibana: Create index pattern `filebeat-*`, visualize Modbus logs (e.g., query `modbus.function_code`).

### 5.2: Operational Procedures
- Run operations in Caldera to simulate attacks.
- View logs in Kibana under Discover.
- Stop: `docker compose down`.
- Logs: Zeek in `./zeek_logs` (host-mapped via volume).

**Pitfall Resolution:**
- If Caldera fails to start: Check logs (`docker logs caldera`)—ensure plugin submodule pulled.
- No logs in ELK: Verify Filebeat paths match Zeek log location; restart services.
- Traffic not on `lo`: Ensure services communicate via 127.0.0.1.

This Dockerized setup provides a portable, reproducible lab. For advanced networking, consider macvlan instead of host mode.