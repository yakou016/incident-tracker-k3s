# Incident Tracker: Docker Compose and K3s

A systems administration coursework project by Anthony Yakoub, University of Minnesota.

This repository packages an incident tracker as a Flask web service backed by PostgreSQL. It includes Docker Compose configuration and Kubernetes manifests for a single-node K3s lab on an Ubuntu Server virtual machine.

## Project scope and contribution

The Flask application began as a course starter application. My lab work focused on container packaging, environment configuration, Compose orchestration, conversion to Kubernetes with Kompose, deployment, and troubleshooting. This portfolio copy preserves the week-4 application and deployment files from my original course repository, with reorganized documentation and a consistent example configuration.

This is a local learning environment. It is not a production deployment, and the repository does not claim cloud hosting, high availability, or an automated CI/CD pipeline.

## How it works

- The browser connects to the Flask web service on port 8080.
- Flask creates, lists, and updates incident tickets in PostgreSQL.
- Compose starts the web container after the database health check succeeds.
- Kubernetes Services connect the workloads. A PersistentVolumeClaim stores PostgreSQL data.
- Database passwords and the Flask secret are supplied locally through an ignored .env file and a Kubernetes Secret.

The included application exposes /health. That route checks the web process only; the home page also reads from the database.

## Run with Docker Compose

Requirements: Docker Engine and Docker Compose v2.

From the repository root:

```bash
cp .env.example .env
# Edit .env and replace both secret placeholders.
docker compose up --build -d
docker compose ps
curl --fail http://localhost:8080/health
curl --fail http://localhost:8080/
```

Open http://localhost:8080 to create a ticket and change its status.

```bash
docker compose down
```

The named database volume is retained by that command. Avoid deleting it if you want to retain your lab tickets.

## Run with K3s

Requirements: a running K3s cluster, kubectl access to it, Docker to build the image, and a default storage class. These instructions assume Docker and K3s run on the same lab machine and the Kubernetes context points to that cluster.

1. Prepare .env as above. Keep POSTGRES_USER=incident_app and POSTGRES_DB=incidents, because the included Kubernetes manifests use those names.
2. If Compose is running, stop it with docker compose down. Compose and K3s use separate database volumes; existing Compose tickets are not migrated.
3. Build the image and import it into K3s's containerd image store:

```bash
docker build -t docker.io/library/week-4-web:latest ./app
docker save docker.io/library/week-4-web:latest -o incident-tracker-image.tar
sudo k3s ctr images import incident-tracker-image.tar
rm incident-tracker-image.tar
```

4. Create or update the lab Secret, apply the manifests, and wait for the database before starting the web deployment:

```bash
kubectl create secret generic db-credentials --from-env-file=.env --dry-run=client -o yaml | kubectl apply -f -
kubectl apply -f k8s/db-data-persistentvolumeclaim.yaml
kubectl apply -f k8s/env-configmap.yaml
kubectl apply -f k8s/db-service.yaml
kubectl apply -f k8s/web-service.yaml
kubectl apply -f k8s/db-deployment.yaml
kubectl rollout status deployment/db --timeout=120s
kubectl apply -f k8s/web-deployment.yaml
kubectl rollout status deployment/web --timeout=120s
kubectl get pods
kubectl get services
kubectl get pvc
```

5. In a separate terminal, forward the web service:

```bash
kubectl port-forward service/web 8080:8080
```

Then open http://localhost:8080 or run the curl checks above. This binds to localhost. For a VM setup, SSH forwarding can expose it locally without binding the service on every interface.

## Troubleshooting example

My web pod entered ImagePullBackOff even though the image was available in Docker. K3s used containerd, which had a separate image store. Importing the image into K3s, matching the Deployment image name, and using imagePullPolicy: IfNotPresent allowed the workload to start.

See [the troubleshooting notes](docs/troubleshooting.md) for the diagnostic commands and what this taught me about layered troubleshooting.

## Limitations

- The app uses Flask's development server and has no authentication or authorization.
- /health does not validate database connectivity.
- The database and web deployments each have one replica; this is not a high-availability design.
- The image uses a local latest tag. Rebuilding it requires importing the replacement and restarting the web deployment.
- The Kubernetes files were generated with Kompose and adjusted for the lab; they are not a hardened production template.

## Repository contents

| Path | Purpose |
| --- | --- |
| app/ | Flask application, templates, Dockerfile, dependencies |
| docker-compose.yml | Web/database orchestration and health checks |
| k8s/ | Kubernetes deployments, services, ConfigMap, and persistent storage |
| .env.example | Placeholder configuration; real values stay local |
| docs/troubleshooting.md | Image-store incident and verification approach |

## Source and attribution

Source: my private INET4031 course repository, yakou016/inet4031, week-4 directory. Application code includes course starter material. This export documents the system administration and deployment work rather than claiming the entire application was independently authored. No new software license is assigned to the starter material.
