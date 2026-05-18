# Docker compose brief

Are you tired by executing `docker run`, `docker build`? so are we.

[Docker Compose](https://docs.docker.com/compose/) is a tool for defining and running multi-container Docker applications.
With Compose, you use a YAML file to configure your application's services. 
Then, with a single command, you create and start all the services from your configuration.

A `docker-compose.yaml` looks like this:

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


# Exercises

## :pencil2: YoloService + YoloFrontend + Monitoring Stack

In this exercise you will compose a full multi-service application:

- **YoloService** - the object detection backend (you've built the image yourself).
- **YoloFrontend** - a web UI that talks to YoloService (you'll build this image yourself too).
- **Prometheus** - scrapes metrics from YoloService.
- **Grafana** - visualises the Prometheus metrics.

The final network architecture looks like the following - you will create two custom networks and connect the services accordingly:

```
┌─────────────────────────────┐     ┌──────────────────────────────┐
│        yolo-net             │     │        monitoring-net        │
│                             │     │                              │
│  yolo-frontend ──► yolo     │     │  prometheus ──► grafana      │
│                   service   │◄────│  (prometheus also joins      │
└─────────────────────────────┘     │   yolo-net to scrape metrics)│
                                    └──────────────────────────────┘
```

#### Notes

- Source code for the YoloService: https://github.com/alonitac/YoloService
- Source code for the YoloFrontend: https://github.com/alonitac/YoloFrontend. You should create a Dockerfile for it (get help from ChatGPT if you need). Then build and push the image to DockerHub, just like you did for YoloService.

- Grafana should persist data in `/var/lib/grafana`, so you should mount a volume for it. - Prometheus should also persist data in `/prometheus` and read its config from `./prometheus.yml`, so you should mount two volumes for it.



At the end, you should open `http://<your-ec2-ip>:3000` in your browser and send an image through the YoloFrontend UI.

In addition, you should be able to restart the stack and confirm data survives:

```bash
docker compose down        # stops and removes containers, but NOT volumes
docker compose up -d
```

Then open Grafana - your data source configuration should still be there.


