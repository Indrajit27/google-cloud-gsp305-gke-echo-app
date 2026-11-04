# GSP305: Deploy a GKE Echo Application

A Google Cloud Skills Boost challenge lab demonstrating how to build a container image, publish it to Google Container Registry (GCR), deploy it to Google Kubernetes Engine (GKE), expose it with a public LoadBalancer, and scale the application to two replicas.

## Lab objectives

- Inspect a Cloud Storage bucket and download the supplied application archive.
- Build the updated Echo application image and tag it as `v2`.
- Push the image to `gcr.io`.
- Deploy the application to a GKE cluster.
- Expose the application externally on port 80, targeting container port 8000.
- Scale the Deployment to two replicas.
- Verify the application through its external IP address.

## Environment

| Item | Value |
|---|---|
| Lab | GSP305 – Deploy GKE Echo App |
| Google Cloud project | Assigned dynamically by Skills Boost |
| GKE cluster | `echo-cluster` |
| Zone | `us-central1-c` (this lab session) |
| Kubernetes Deployment | `echo-web` |
| Kubernetes Service | `echo-web` |
| Container image | `gcr.io/PROJECT_ID/echo-app:v2` |
| Service type | `LoadBalancer` |
| Service port | `80` |
| Container target port | `8000` |
| Desired replicas | `2` |

> **Note:** The project ID and zone are assigned by the lab. Replace the example values with the values shown in your own lab session.

## 1. Set the project and connect to GKE

Get the active project ID:

```bash
PROJECT_ID=$(gcloud config get-value project)
echo "$PROJECT_ID"
```

Connect to the existing cluster (replace the zone if your lab uses a different one):

```bash
gcloud container clusters get-credentials echo-cluster --zone=us-central1-c
kubectl get nodes
```

The cluster node should report `Ready`.

## 2. Download and extract the application

List the lab bucket and locate `echo-web-v2.tar.gz`:

```bash
gcloud storage buckets list
gcloud storage ls gs://BUCKET_NAME/
```

Download and extract the archive (replace `BUCKET_NAME`):

```bash
gcloud storage cp gs://BUCKET_NAME/echo-web-v2.tar.gz .
tar -xzf echo-web-v2.tar.gz
ls -la
```

The extracted application context includes a `Dockerfile` and Go application source (`main.go`).

## 3. Build and tag the Docker image

```bash
docker build -t gcr.io/$PROJECT_ID/echo-app:v2 .
docker images
```

Confirm that `gcr.io/PROJECT_ID/echo-app:v2` appears in the local image list.

## 4. Push the image to Container Registry

Configure Docker authentication:

```bash
gcloud auth configure-docker
```

Push the image:

```bash
docker push gcr.io/$PROJECT_ID/echo-app:v2
```

Verify the published tag:

```bash
gcloud container images list-tags gcr.io/$PROJECT_ID/echo-app
```

The image should have the `v2` tag.

## 5. Deploy the application

Create the Kubernetes Deployment using the v2 image:

```bash
kubectl create deployment echo-web --image=gcr.io/$PROJECT_ID/echo-app:v2
```

Expose it through a public LoadBalancer Service on port 80, forwarding to application port 8000:

```bash
kubectl expose deployment echo-web \
  --type=LoadBalancer \
  --port=80 \
  --target-port=8000
```

Check the Deployment and Service:

```bash
kubectl get deployments
kubectl get services
```

The external IP may initially show `<pending>` while Google Cloud provisions the load balancer.

## 6. Scale to two replicas

```bash
kubectl scale deployment echo-web --replicas=2
kubectl get deployments
kubectl get pods -o wide
```

Expected state:

- Deployment desired replicas: 2
- Ready replicas: 2
- Both Echo application pods: `Running`

## 7. Verify the application

Wait until the Service has an external IP:

```bash
kubectl get service echo-web
```

Retrieve the IP and send an HTTP request:

```bash
EXTERNAL_IP=$(kubectl get service echo-web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "$EXTERNAL_IP"
curl "http://$EXTERNAL_IP"
```

A successful response should include the application name and version information, for example:

```text
Echo Test
Version: 2.0.0
Hostname: echo-web-<pod-id>
Host ip-address(es): 10.x.x.x
```

The hostname can differ between requests because the LoadBalancer Service distributes traffic across the available pods.

## Architecture

```text
Cloud Storage
  echo-web-v2.tar.gz
        |
        v
Cloud Shell: Docker build
  gcr.io/PROJECT_ID/echo-app:v2
        |
        v
Google Container Registry
        |
        v
GKE cluster: echo-cluster
        |
        v
Deployment: echo-web
  2 replicas
        |
        v
Service: LoadBalancer
  External TCP port 80
        |
        v
Echo application container port 8000
```

## Skills demonstrated

- Google Kubernetes Engine (GKE)
- Kubernetes Deployments, Pods, Services and replica scaling
- Docker image builds and tagging
- Google Container Registry (GCR)
- Cloud Storage CLI
- Kubernetes rollout and workload verification
- External LoadBalancer exposure and HTTP testing

## Lab completion evidence

In this lab session, the application returned:

```text
Echo Test
Version: 2.0.0
Hostname: echo-web-5b46f84c55-zflcf
Host ip-address(es): 10.104.0.15
```

Both application pods were observed in the `Running` state with `1/1` containers ready.

> This repository documents the hands-on lab implementation. It does not contain the Google Cloud Skills Boost scoring system or claim an independently verified certification result. Use the lab's **Check my progress** controls to confirm the score in your own Skills Boost session.

## Cleanup

If you are working in your own Google Cloud project (not a temporary Skills Boost environment), remove the Service and Deployment when finished:

```bash
kubectl delete service echo-web
kubectl delete deployment echo-web
```

---

**Lab:** Google Cloud Skills Boost – GSP305  
**Focus:** Container image build, registry publishing, GKE deployment, scaling and service verification.
