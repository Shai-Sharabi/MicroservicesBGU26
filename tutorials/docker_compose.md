# Docker compose brief

Are you tired by executing `docker run`, `docker build`? so are we.

[Docker Compose](https://docs.docker.com/compose/) is a tool for defining and running multi-container Docker applications.
With Compose, you use a YAML file to configure your application's services. 
Then, with a single command, you create and start all the services from your configuration.

Using Compose is essentially a three-step process:

1. Define your app's environment with a `Dockerfile` so it can be reproduced anywhere.
2. Define the services that make up your app in `docker-compose.yml` so they can be run together in an isolated environment.
3. Run `docker compose up` and the Docker compose command starts and runs your entire app.

A `docker-compose.yml` looks like this:

```yaml
services:
  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
```

The given Docker Compose file describes a multi-service application with two services: `prometheus` and `grafana`. 

## Compose benefits 

Using Docker Compose offers several benefits:

- **Simplified Container Orchestration**: Docker Compose allows for the definition and management of multi-container applications as a single unit.
- **Reproducible Environments**: Since compose is defined in a YAML file, it's easy to deploy the same environment in different machine without missing any `docker run` command. This ensures that the application runs consistently across different machines.
- **Automate Volumes and Networking**: Docker Compose automatically creates a network for the application and assigns a unique DNS name to each service. No need to create networks and volumes. 

[docker_compose_netflix_n_monitoring]: https://exit-zero-academy.github.io/DevOpsTheHardWayAssets/img/docker_compose_netflix_n_monitoring.png

# Exercises

## :pencil2: YoloService + Frontend + Monitoring Stack

In this exercise you will compose a full multi-service application:

- **YoloService** – the object detection backend (you'll build the image yourself).
- **YoloFrontend** – a web UI that talks to YoloService (you'll build this image yourself too).
- **Prometheus** – scrapes metrics from YoloService.
- **Grafana** – visualises the Prometheus metrics.

The final architecture looks like this:

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│        yolo-net             │     │        monitoring-net         │
│                             │     │                               │
│  yolo-frontend ──► yolo     │     │  prometheus ──► grafana       │
│                   service   │◄────│  (prometheus also joins       │
└─────────────────────────────┘     │   yolo-net to scrape metrics) │
                                    └──────────────────────────────┘
```

### Step 1 – Build the images

Clone the source repositories and build both images locally.

**YoloService:**

```bash
git clone https://github.com/alonitac/YoloService.git
cd YoloService
docker build -t yolo-service:0.0.1 .
cd ..
```

**YoloFrontend** (check the repo for its Dockerfile location):

```bash
git clone https://github.com/alonitac/YoloFrontend.git
cd YoloFrontend
docker build -t yolo-frontend:0.0.1 .
cd ..
```

### Step 2 – Create the Prometheus config file

Prometheus needs to know which service to scrape.
Because Compose puts all services on a shared DNS, you can use the service name directly as the hostname.

Create `prometheus.yml` next to your `docker-compose.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'yolo-service'
    static_configs:
      - targets: ['yolo-service:8080']
```

### Step 3 – Write the `docker-compose.yml`

Requirements to fulfil:

| Requirement | Detail |
|---|---|
| Two custom networks | `yolo-net` (YoloService + YoloFrontend) and `monitoring-net` (Prometheus + Grafana) |
| Prometheus bridges both networks | It must join `yolo-net` so it can reach YoloService's `/metrics` endpoint |
| Persistent data via **volumes** | YoloService (`/app/uploads/`, `/app/predictions.db`), Prometheus (`/prometheus`), Grafana (`/var/lib/grafana`) |
| Ports exposed | YoloFrontend → `8080`, Prometheus → `9090`, Grafana → `3000` |

Your `docker-compose.yml` should follow this skeleton – fill in the missing pieces:

```yaml
networks:
  yolo-net:
  monitoring-net:

volumes:
  yolo-uploads:
  yolo-db:
  prometheus-data:
  grafana-data:

services:
  yolo-service:
    image: yolo-service:0.0.1
    networks:
      - yolo-net
    volumes:
      - yolo-uploads:/app/uploads/
      - yolo-db:/app/predictions.db

  yolo-frontend:
    image: yolo-frontend:0.0.1
    ports:
      - "8080:80"
    networks:
      - yolo-net
    # hint: the frontend needs to know where YoloService is – check the repo
    # for the environment variable name

  prometheus:
    image: prom/prometheus
    ports:
      - "9090:9090"
    networks:
      - yolo-net        # to scrape yolo-service
      - monitoring-net  # to be queried by grafana
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus

  grafana:
    image: grafana/grafana
    ports:
      - "3000:3000"
    networks:
      - monitoring-net
    volumes:
      - grafana-data:/var/lib/grafana
```

### Step 4 – Bring the stack up

```bash
docker compose up -d
```

Verify all four containers are running:

```bash
docker compose ps
```

### Step 5 – Verify

1. Open `http://<your-ec2-ip>:8080` in your browser and send an image through the YoloFrontend UI.
2. Open `http://<your-ec2-ip>:9090` and run the query `http_requests_total` – you should see data from YoloService.
3. Open `http://<your-ec2-ip>:3000` (default credentials: `admin` / `admin`), add Prometheus as a data source using the URL `http://prometheus:9090`, and build a simple graph.

### Step 6 – Verify data persistence

Restart the stack and confirm data survives:

```bash
docker compose down        # stops and removes containers, but NOT volumes
docker compose up -d
```

- Send another prediction through YoloFrontend – previous uploads should still be in the database.
- Open Grafana – your data source configuration should still be there.

> **Why `docker compose down` instead of `docker compose down -v`?**
> The `-v` flag removes named volumes together with the containers. Without it, volumes are kept, which is what you want for persistent data.

