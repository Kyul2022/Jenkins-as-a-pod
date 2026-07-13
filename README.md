# Jenkins as a pod on Kubernetes

This guide walks through running Jenkins on Kubernetes, giving it permission to manage its own pods via a `ServiceAccount`, and using the Kubernetes plugin so Jenkins can spin up ephemeral agent pods for CI/CD.

## 1. Create a dedicated namespace

Good practice: keep everything related to Jenkins (pods, services, volumes) isolated in its own namespace.

```bash
kubectl create namespace jenkins
```

## 2. Create the deployment

Create the deployment manifest:

```bash
nano jenkins.yaml
```

This will define the Jenkins pod, its container image, ports, and (later) its volume and service account.

## 3. Grant Jenkins permission to manage pods (ServiceAccount)

Since Jenkins needs to create and manage pods through the Kubernetes API, its pod needs the right permissions. The pod and the Jenkins container inside it share the same permissions, so the clean, production-grade way to do this is through a `ServiceAccount` bound to a `ClusterRole`.

### 3.1 Create the ServiceAccount

```bash
kubectl create serviceaccount jenkins-agent -n jenkins
```

### 3.2 Bind it to a role

Kubernetes ships with predefined roles (like `admin`). Bind the ServiceAccount to one:

**Namespace-scoped** (access limited to the `jenkins` namespace):

```bash
kubectl create rolebinding jenkins-agent-admin \
  --clusterrole=admin \
  --serviceaccount=jenkins:jenkins-agent \
  --namespace=jenkins
```

**Cluster-scoped** (access across all namespaces):

```bash
kubectl create clusterrolebinding jenkins-agent-admin \
  --clusterrole=admin \
  --serviceaccount=jenkins:jenkins-agent
```

### 3.3 Attach the ServiceAccount to the deployment

In the `jenkins.yaml` deployment, add the following under `spec`:

```yaml
serviceAccountName: jenkins
```

More configuration details will come later in the Jenkinsfile.

## 4. Store Docker Hub credentials as a secret

Agent pods are ephemeral, so the resulting image needs to be pushed somewhere persistent. Create a secret holding the Docker Hub credentials (via command, not a file, to avoid leaking secrets in version control):

```bash
kubectl create secret docker-registry docker-creds \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username=<username> \
  --docker-password=<password> \
  --docker-email=<email> \
  -n jenkins
```

Notes:
- `docker-registry` is a special Kubernetes secret type made for this exact use case.
- `https://index.docker.io/v1/` is the actual registry endpoint — `hub.docker.com` is just the web portal.

This secret is later mounted into the agent pod as a configuration file so it can push images.

## 5. Persistent storage for the Jenkins pod

Jenkins needs a volume to persist its data (jobs, config, plugins) across restarts.

1. Create a `PersistentVolumeClaim` (PVC), specifying the size and storage class you need.
2. You can create a matching `PersistentVolume` yourself, or let Kubernetes provision one automatically if none match the PVC's requirements.
3. In the deployment, under the container section where the Jenkins image is defined, add a `volumeMounts` entry pointing to the volume backed by the PVC.

## 6. Kubernetes plugin for dynamic agents

Install the Kubernetes plugin in Jenkins so it can use ephemeral Kubernetes pods as build agents instead of static agents. Once configured, Jenkins will:

1. Spin up a temporary pod as a build agent when a job runs.
2. Run the build/test/package steps inside that pod.
3. Push the resulting image to Docker Hub using the mounted credentials.
4. Tear down the agent pod once the job completes.
5. Optionally deploy the application back onto the same cluster.

## Summary

| Component | Purpose |
|---|---|
| `jenkins` namespace | Isolates all Jenkins-related resources |
| Jenkins deployment + service | Runs Jenkins itself as a pod |
| PVC + volume | Persists Jenkins data across restarts |
| ServiceAccount + RoleBinding/ClusterRoleBinding | Lets Jenkins create/manage pods via the Kubernetes API |
| `docker-creds` secret | Lets ephemeral agent pods push images to Docker Hub |
| Kubernetes plugin | Lets Jenkins use ephemeral pods as CI/CD agents |
