# local-services

A collection of local infrastructure services for development, running via Docker Compose. This setup uses the Host Networking approach for Kafka to allow seamless communication across different Docker Compose projects without manual network management.

## Quick Start

1. **Start the services**:

   For standard local development (FastAPI and Kafka on the same machine):
   ```bash
   docker compose up -d
   ```

   To allow access from **other PCs** in your local network:
   ```bash
   HOST_IP=$(hostname -I | awk '{print $1}') docker compose up -d
   ```

2. **Verify**:
   ```bash
   docker compose ps
   ```

## Services

| Service | Image | Access (from Host) | Access (from other Containers) | Persistence |
|---------|-------|--------------------|--------------------------------|-------------|
| **Redis** | `redis:8-alpine` | `localhost:6000` | Same Compose: `redis:6379` | AOF enabled |
| **Kafka** | `apache/kafka:latest` | `localhost:9092` | **`172.17.0.1:9092`** | `kafka_data` volume |

- **Redis** remains on a standard bridge network, mapped to host port `6000`.
- **Kafka** runs in `network_mode: host`. This removes Docker network isolation, allowing it to behave like a native process on your host machine.

## Network Architecture

This project avoids "External Docker Networks" to simplify cross-project communication.
- **Kafka** is bound directly to the host's network stack.
- **FastAPI/Other Containers** reach Kafka through the default Docker bridge gateway (**`172.17.0.1`** on Linux).

### Why this approach?
By using `network_mode: host` and advertising the Docker bridge IP, you don't need to:
1. Pre-create networks (`docker network create ...`).
2. Hardcode your local LAN IP address.
3. Modify the `networks` section of every microservice that wants to talk to Kafka.

## Connecting to Kafka from Other Docker Compose Projects

If your FastAPI service (or any other container) is in a different `docker-compose.yml`, it can reach Kafka automatically.

**In your application configuration:**
```yaml
environment:
  # 172.17.0.1 is the default Linux Docker gateway to the host
  - KAFKA_BOOTSTRAP_SERVERS=172.17.0.1:9092
```

**In your Python code:**
```python
# Connect to the host machine's port 9092 from inside the container
bootstrap_servers = "172.17.0.1:9092"
```

## Access from Another PC
If a teammate needs to connect to your Kafka broker from a different computer:
1. Start the services using the `HOST_IP` command mentioned in the **Quick Start**.
2. They should connect using your physical IP: `your-ip-address:9092`.

## Persistence

Data is stored in named Docker volumes to ensure it survives restarts:
- `redis_data` – Redis AOF (Append Only File).
- `kafka_data` – Kafka logs, metadata, and KRaft controller state.

To completely wipe all data:
```bash
docker compose down -v
```

## Troubleshooting (Linux)

- **Cannot reach Kafka from container:** Verify your Docker gateway IP by running `ip addr show docker0`. It is usually `172.17.0.1`. If yours is different, set `HOST_IP` to your specific gateway IP before running `docker compose up`.
- **Port Conflicts:** Since Kafka uses `network_mode: host`, ensure port `9092` and `9093` are not being used by any other service on your host machine.
- **Firewall:** If connecting from another PC, ensure your Linux firewall (ufw/iptables) allows incoming traffic on port `9092`.
