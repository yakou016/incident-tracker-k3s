# Troubleshooting K3s image availability

## Observed issue

The database pod started, but the web pod entered ImagePullBackOff. The web application image existed in Docker, which initially made it appear that the cluster should be able to run it.

## Investigation and fix

K3s uses containerd. Docker and K3s do not automatically share local images.

Useful checks:

```bash
kubectl get pods
kubectl describe deployment/web
kubectl get events --sort-by=.metadata.creationTimestamp
docker images
sudo k3s ctr images list
```

Inspect the affected pod's events to identify the failed pull. Build and export the application image with the exact name referenced in k8s/web-deployment.yaml, then import it into K3s as described in the README.

The included Deployment uses:

```yaml
image: docker.io/library/week-4-web:latest
imagePullPolicy: IfNotPresent
```

This allows K3s to use its local image. A missing image still triggers a registry pull, so the import and name must both be correct.

## Verification

Check the database rollout, web rollout, storage claim, and HTTP response separately:

```bash
kubectl rollout status deployment/db --timeout=120s
kubectl rollout status deployment/web --timeout=120s
kubectl get pods
kubectl get pvc
curl --fail http://localhost:8080/health
curl --fail http://localhost:8080/
```

The HTTP checks require the port-forwarding terminal to remain open.

## What I learned

A running VM does not prove that Kubernetes or the application is healthy. Likewise, an image in Docker does not prove that the runtime used by Kubernetes can access it. Troubleshooting requires following the request across the server, cluster, image source, service, and database.

These notes describe the lab incident documented in my September 30 coursework reflection. The portfolio export has not been deployed in this workspace.
