# local-services

A collection of local infrastructure services for development, running via Docker Compose.

## Quick Start

1. **Create the shared network** (required only once):

    ```bash
    docker network create kafka-net
    ```

2. **Start the services**:

    ```bash
    docker compose up -d
    ```

   Use `docker compose up --build` if you need to rebuild custom images (currently none).

3. **Verify**:

    ```bash
    docker compose ps
    ```

## Services

| Service | Image            | Host Port | Internal Port | Persistence | Access                                                                 |
|---------|------------------|-----------|---------------|-------------|------------------------------------------------------------------------|
| Redis   | `redis:8-alpine` | `6000`    | `6379`        | AOF enabled | Host: `localhost:6000`<br>Same Compose: `redis:6379`                   |
| Kafka   | `apache/kafka:latest` | `9092` | `29092` (internal) | Yes (volume `kafka_data`) | Host: `localhost:9092`<br>Other Compose (on `kafka-net`): `kafka-broker:29092` |

- **Redis** uses AOF persistence (`appendonly yes`), data stored in the `redis_data` volume.
- **Kafka** runs in KRaft mode (no ZooKeeper), with dual listeners to support both local host tools and other Docker containers.

## Network Architecture

- **Default bridge network**: Redis is attached here; it’s isolated from the outside unless you explicitly connect other Compose files to it.
- **External network `kafka-net`**: Kafka broker is attached to this manually created network. This network is **shared across Compose projects** so other services (like your FastAPI gateway) can reach Kafka without being in the same Compose file.

> **Why an external network?**  
> Kafka is often consumed by multiple micro‑services in separate Compose stacks. By placing the broker on a pre‑created external network, any other Compose project can join `kafka-net` and communicate with the broker using the hostname `kafka-broker`.

## Connecting to Kafka from Other Docker Compose Projects

1. **Create the external network** (once, as shown above).
2. **In the other project’s `docker-compose.yml`**, declare and attach the network:

    ```yaml
    networks:
      kafka-net:
        external: true

    services:
      your-service:
        # ...
        networks:
          - default  # keep if you need local communication
          - kafka-net  # essential to connect to Kafka
    ```

3. **Use the internal listener address** in your application configuration:

    ```yaml
    environment:
      - KAFKA_BOOTSTRAP_SERVERS=kafka-broker:29092
    ```

   Or directly in code:

    ```python
    bootstrap_servers = "kafka-broker:29092"
    ```

**Why different ports?**  
- `localhost:9092` – advertised to host tools (KafkIO, `kafka-console-producer`).  
- `kafka-broker:29092` – advertised to Docker containers on the `kafka-net` network.  
A client receives the advertised address from the broker; using the correct listener avoids connection refused errors inside containers.

## Connecting to Redis from Other Containers

If another service needs Redis, you can either:
- Add that service to the same `docker-compose.yml` and use the hostname `redis:6379`.
- Or attach both projects to a shared network (similar to Kafka) and use the service name. For simplicity, most setups just keep Redis in the same Compose file as its consumers.

## Persistence

Data is stored in named Docker volumes:
- `redis_data` – Redis AOF file.
- `kafka_data` – Kafka logs and metadata.

Volumes survive `docker compose down` (unless you add `-v`). To completely reset state:

    ```bash
    docker compose down -v
    ```

## Useful Commands

**Check Kafka topics** (from host):

    ```bash
    docker exec -it kafka-broker kafka-topics.sh --list --bootstrap-server localhost:9092
    ```

**Produce a test message**:

    ```bash
    docker exec -it kafka-broker kafka-console-producer.sh --topic test --bootstrap-server localhost:9092
    ```

**Consume messages**:

    ```bash
    docker exec -it kafka-broker kafka-console-consumer.sh --topic test --from-beginning --bootstrap-server localhost:9092
    ```

## Troubleshooting

- **Kafka fails to start** – Ensure the external network exists: `docker network create kafka-net`.
- **“No connection to node with id 1”** – Your container is using the wrong bootstrap address. Use `kafka-broker:29092` (internal listener) inside Docker, not `localhost:9092`.
- **“GroupCoordinatorNotAvailableError”** – Likely missing `KAFKA_CLUSTER_ID` or replication factor settings; already configured here.
- **Repeated “Marking the coordinator dead”** – Usually harmless during startup. If persistent, increase consumer timeouts (`session_timeout_ms`, `heartbeat_interval_ms`).
