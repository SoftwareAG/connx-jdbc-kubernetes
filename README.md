# CONNX JDBC Server on Kubernetes and OpenShift: Deployment Guide

This package deploys the CONNX JDBC server (`cnxjdbc64`) on any Kubernetes platform, including Red Hat OpenShift (ROSA, ARO), AKS, EKS and GKE. The server listens on TCP port 7502.

- The CONNX registry (`connxreg64.db`) is supplied as a ConfigMap and mounted **read-only** at `/config`. The pods keep no state.
- Because the pods keep no state, you can run as many replicas as you need. Updates are rolling, with no downtime.
- The manifests default to OpenShift. The pods run under the default `restricted-v2` security context constraint (SCC). You do **not** need to grant `anyuid` or any privileged SCC.
- For AKS, EKS, GKE and other Kubernetes platforms, see [Running on AKS, EKS, GKE and other Kubernetes](#running-on-aks-eks-gke-and-other-kubernetes). It needs one line uncommented.

The commands use `kubectl`. On OpenShift, `oc` accepts the same commands.

The commands are written for a Bash shell on Linux or macOS. On Windows, use Git Bash or WSL, not PowerShell. In Git Bash, run `export MSYS_NO_PATHCONV=1` first. Otherwise Git Bash rewrites container paths such as `/config` to Windows paths and the `exec` commands fail.

> **Disclaimer:** These manifests are a sample, provided as-is and without warranty or support. They are not part of the Software AG product suite and are not covered by Software AG support or maintenance. There is no guarantee that they work in your environment. Review, adapt and test them before use. You run them at your own responsibility. See [License](#license).

---

## Contents

| File | Purpose | Action required |
| --- | --- | --- |
| `00-namespace.yaml` | Namespace (OpenShift project) `connx` | None |
| `01-configmap.yaml` | License acceptance and registry path | Review `ACCEPT_EULA` |
| `02-deployment.yaml` | The CONNX JDBC server | **Required: set `image`.** Non-OpenShift: uncomment `runAsUser` |
| `03-service.yaml` | Service on port 7502 | Optional: change `type` for external clients |
| `.gitignore` | Keeps `connxreg64.db` and image archives out of Git | None |
| `LICENSE` | Apache License 2.0, for the manifests and documentation in this repository | None |

The registry ConfigMap `connx-jdbc-registry` is not a file in this folder. You create it from your `connxreg64.db` (step 2).

The files are numbered so that `kubectl apply -f .` creates them in the correct order.

---

## Prerequisites

- Kubernetes 1.25 or later, or Red Hat OpenShift Container Platform 4.12 or later.
- `kubectl` (or `oc` on OpenShift), logged in to the cluster with a user who can create namespaces.
- The CONNX JDBC server container image, in a container registry the cluster can pull from. Building and pushing the image is described separately.
- A valid CONNX license entitlement.

No persistent storage is needed.

---

## Running on AKS, EKS, GKE and other Kubernetes

The manifests default to OpenShift. On any other Kubernetes platform, make these changes. Everything else in this guide applies unchanged.

**1. Set the pod user (required).** In `02-deployment.yaml`, uncomment `runAsUser`:

```yaml
      securityContext:
        runAsNonRoot: true
        runAsUser: 1724                         # Kubernetes other than OpenShift: uncomment
```

The image user `sagadmin` is not numeric, so without this the pods fail with `CreateContainerConfigError: image has non-numeric user (sagadmin), cannot verify user is non-root`. With it, the pods also meet the Pod Security Standards `restricted` profile. Do **not** set it on OpenShift, which assigns its own UID and rejects a fixed one.

**2. Give the cluster pull access to the image registry:**

| Platform | Registry | Pull access |
| --- | --- | --- |
| AKS | Azure Container Registry | `az aks update -n <CLUSTER> -g <RESOURCE_GROUP> --attach-acr <ACR_NAME>` |
| EKS | Amazon ECR | The node IAM role needs `AmazonEC2ContainerRegistryReadOnly` (managed node groups have it by default) |
| GKE | Artifact Registry | The node service account needs `roles/artifactregistry.reader` |
| Other | Any | A pull secret (see [Step 1](#step-1-set-the-image)) |

With this access in place, leave `imagePullSecrets` commented out.

**3. Skip the OpenShift-only items:** the internal registry address in step 1 and the `restricted-v2` check in step 5. The `openshift.io/*` annotations in `00-namespace.yaml` are ignored on other platforms. You don't need to remove them.

**4. External access (optional).** `type: LoadBalancer` in `03-service.yaml` creates a public load balancer. For a load balancer reachable only from your private network, add the annotation for your platform under `metadata.annotations`:

| Platform | Annotation |
| --- | --- |
| AKS | `service.beta.kubernetes.io/azure-load-balancer-internal: "true"` |
| EKS (AWS Load Balancer Controller) | `service.beta.kubernetes.io/aws-load-balancer-type: external` and `service.beta.kubernetes.io/aws-load-balancer-scheme: internal` |
| GKE | `networking.gke.io/load-balancer-type: "Internal"` |

---

## Step 1: Set the image

In `02-deployment.yaml`, replace the placeholder with the full reference of your CONNX JDBC server image:

```yaml
        image: <REGISTRY>/<REPOSITORY>/connx-jdbc:<TAG>
```

For example:

```yaml
        image: registry.example.com/softwareag/connx-jdbc:14.9.25364
```

If you skip this, the pods fail with `InvalidImageName`. Use a version tag, not `latest`, so that you always know which version is running and can roll back.

**OpenShift internal registry.** If the image is in the internal registry, in the `connx` project, use the **in-cluster** address, not the external route:

```yaml
        image: image-registry.openshift-image-registry.svc:5000/connx/connx-jdbc:<TAG>
```

Pods in the `connx` project can pull from it without a pull secret.

**Registry credentials.** If the registry needs credentials, create a pull secret and uncomment `imagePullSecrets` in `02-deployment.yaml`:

```bash
kubectl apply -f 00-namespace.yaml
kubectl create secret docker-registry connx-registry-pull -n connx \
  --docker-server=<REGISTRY> \
  --docker-username=<USER> \
  --docker-password=<PASSWORD_OR_TOKEN>
```

On OpenShift, if the registry is already in the cluster-wide pull secret (`pull-secret` in `openshift-config`), you do not need a project pull secret. On AKS, EKS and GKE, see [pull access](#running-on-aks-eks-gke-and-other-kubernetes).

---

## Step 2: Create the registry ConfigMap

The server reads its registry from the ConfigMap `connx-jdbc-registry`. The pods mount it read-only and never change it.

If you do not have a registry file yet, copy the default one out of the image with a temporary pod. Use the same image as in step 1:

```bash
kubectl apply -f 00-namespace.yaml
kubectl run connx-registry-copy -n connx --image=<REGISTRY>/<REPOSITORY>/connx-jdbc:<TAG> \
  --restart=Never --command -- sleep 600
kubectl wait pod/connx-registry-copy -n connx --for=condition=Ready --timeout=120s
kubectl exec connx-registry-copy -n connx -- cat /opt/softwareag/connx/connxreg64.db > connxreg64.db
kubectl delete pod connx-registry-copy -n connx
```

`kubectl run` may print a `PodSecurity "restricted"` warning. The warning is harmless for this short-lived pod. If you use a pull secret, the temporary pod also needs it: add `--overrides='{"spec":{"imagePullSecrets":[{"name":"connx-registry-pull"}]}}'`.

Create the ConfigMap from the file:

```bash
kubectl create configmap connx-jdbc-registry -n connx --from-file=connxreg64.db --save-config
```

`--save-config` lets you update the ConfigMap later with `kubectl apply` (see [Updating the registry](#updating-the-registry)). The registry is a binary file of a few KB. A ConfigMap holds up to 1 MiB.

Keep `connxreg64.db` in your configuration store. That file, not the cluster, is your copy of record. It can contain connection details, so `.gitignore` keeps it out of this folder's Git repository.

---

## Step 3: Configure

Go through this checklist before deploying.

1. **Image (required).** Set in step 1.
2. **Platform.** On OpenShift, no change is needed. On AKS, EKS, GKE or other Kubernetes, uncomment `runAsUser: 1724` in `02-deployment.yaml`. See [Running on AKS, EKS, GKE and other Kubernetes](#running-on-aks-eks-gke-and-other-kubernetes).
3. **License.** `01-configmap.yaml` sets `ACCEPT_EULA: "Y"`. This value confirms that you accept the Software AG license terms for your CONNX entitlement. Without it the server does not start.
4. **Replicas (optional).** `02-deployment.yaml` runs 2 replicas. Change `replicas` as needed.
5. **Namespace (optional).** To deploy somewhere other than `connx`, replace `connx` in `00-namespace.yaml`, in the `namespace:` field of every other file, and in the commands. If you use the OpenShift internal registry, the image path must match the new project.
6. **External access (optional).** See [Connecting clients](#connecting-clients).

To have the cluster validate the manifests without creating anything:

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f . --dry-run=server
```

The first command creates the namespace, because the namespaced files cannot be validated until it exists.

---

## Step 4: Deploy

From the folder that contains these files:

```bash
kubectl apply -f .
kubectl rollout status deploy/connx-jdbc -n connx
```

---

## Step 5: Verify

```bash
# All pods are Running and 1/1 Ready
kubectl get pods -n connx -l app=connx-jdbc

# The log ends with: "Server starting to Wait on new Connections."
# The warnings "environment setting LANG is not set" and "LC_ALL is not set" are harmless.
kubectl logs deploy/connx-jdbc -n connx

# The registry is mounted from the ConfigMap
kubectl exec deploy/connx-jdbc -n connx -- ls -lL /config

# OpenShift only: the pods were admitted under restricted-v2 (expected output: restricted-v2)
kubectl get pods -n connx -l app=connx-jdbc -o jsonpath='{.items[0].metadata.annotations.openshift\.io/scc}{"\n"}'
```

To test from your workstation without exposing the service:

```bash
kubectl port-forward svc/connx-jdbc 7502:7502 -n connx
```

Then point a JDBC client at `localhost:7502`.

---

## Connecting clients

| Client location | Address |
| --- | --- |
| Same cluster | `connx-jdbc.connx.svc.cluster.local:7502` |
| Outside the cluster | Requires a `LoadBalancer` or `NodePort` service (below) |

JDBC is a plain TCP protocol, so an HTTP ingress or an OpenShift Route cannot be used.

To expose the server, edit `03-service.yaml`:

- **`type: LoadBalancer`** (recommended) on cloud platforms (ROSA, ARO, AKS, EKS, GKE), or on-premises with MetalLB.
- **`type: NodePort`** on clusters without a load balancer. Clients connect to any node IP on the assigned port (30000 to 32767).

Then apply the change and get the address:

```bash
kubectl apply -f 03-service.yaml
kubectl get svc connx-jdbc -n connx             # EXTERNAL-IP, or the NodePort in PORT(S)
kubectl get nodes -o wide                       # Node IPs, for NodePort
```

The service spreads **connections** across the pods. A JDBC session stays on the pod it connected to.

Restrict network access to port 7502 to known client networks, for example with a `NetworkPolicy`, cloud security groups, or firewall rules.

---

## Updating the registry

Edit `connxreg64.db` outside the cluster, then replace the ConfigMap and restart the pods:

```bash
kubectl create configmap connx-jdbc-registry -n connx --from-file=connxreg64.db \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl rollout restart deploy/connx-jdbc -n connx
kubectl rollout status deploy/connx-jdbc -n connx
```

- The server reads the registry when it starts, so a change takes effect only after the restart. Restart straight after updating the ConfigMap. Otherwise a pod that restarts for another reason picks up the new registry while the others still use the old one.
- The restart is rolling: each new pod becomes ready before an old one stops. JDBC sessions on a stopped pod are closed, and clients must reconnect.

To get the registry currently in the cluster:

```bash
kubectl get configmap connx-jdbc-registry -n connx \
  -o jsonpath='{.binaryData.connxreg64\.db}' | base64 -d > connxreg64.db
```

Windows PowerShell 5.1 changes binary output when it is redirected. Run this from Bash, Git Bash or WSL.

---

## Scaling

```bash
kubectl scale deploy/connx-jdbc -n connx --replicas=<N>
```

To keep the change, also set `replicas` in `02-deployment.yaml`. Otherwise the next `kubectl apply -f .` resets it.

---

## Upgrading

1. Make the new image version available in your registry.
2. Update `image:` in `02-deployment.yaml`.
3. Apply and watch the rollout:

```bash
kubectl apply -f 02-deployment.yaml
kubectl rollout status deploy/connx-jdbc -n connx
```

The update is rolling, with no downtime. The registry ConfigMap is not changed.

To roll back, set the previous image tag in `02-deployment.yaml` and apply again, or run `kubectl rollout undo deploy/connx-jdbc -n connx`.

---

## Uninstalling

```bash
kubectl delete -f .
```

> **Warning:** This deletes the namespace, including the registry ConfigMap and, on OpenShift, any images pushed to the internal registry under `connx`. Keep your copy of `connxreg64.db` (see [Updating the registry](#updating-the-registry)).

To remove the server but keep the namespace and registry, delete only the workload:

```bash
kubectl delete -f 02-deployment.yaml -f 03-service.yaml
```

---

## Troubleshooting

| Symptom | Cause | Fix |
| --- | --- | --- |
| Pod status `InvalidImageName` | Image placeholder not replaced | Set `image:` in `02-deployment.yaml` (step 1) |
| Pod status `ImagePullBackOff` / `ErrImagePull` | Wrong image reference, or the registry needs credentials | Check the reference with `kubectl describe pod -n connx -l app=connx-jdbc`. For the OpenShift internal registry, use the `image-registry.openshift-image-registry.svc:5000` address, not the route. Otherwise create a pull secret (step 1). |
| Pod stays `ContainerCreating`, event `configmap "connx-jdbc-registry" not found` | The registry ConfigMap was not created | Create it (step 2) |
| Pod status `CreateContainerConfigError`: `image has non-numeric user (sagadmin), cannot verify user is non-root` | Non-OpenShift Kubernetes with `runAsUser` commented out | Uncomment `runAsUser: 1724` in `02-deployment.yaml` (see [Running on AKS, EKS, GKE and other Kubernetes](#running-on-aks-eks-gke-and-other-kubernetes)) |
| OpenShift: `pods "connx-jdbc-..." is forbidden: unable to validate against any security context constraint` | `runAsUser` (or `fsGroup`) is set to a fixed value outside the project range | Comment out `runAsUser` in `02-deployment.yaml`. OpenShift assigns the UID. |
| Container exits with code 10 | License not accepted | Set `ACCEPT_EULA: "Y"` in `01-configmap.yaml` |
| A registry change has no effect | The pods were not restarted | `kubectl rollout restart deploy/connx-jdbc -n connx` |
| External clients cannot connect | Service is `ClusterIP` (in-cluster only), or a firewall blocks the port | See [Connecting clients](#connecting-clients) |

To diagnose other problems, collect the output of:

```bash
kubectl get all,configmap -n connx
kubectl get events -n connx --sort-by=.lastTimestamp
kubectl describe pod -n connx -l app=connx-jdbc
kubectl logs deploy/connx-jdbc -n connx
```

---

## License

The manifests and documentation in this repository are licensed under the [Apache License 2.0](LICENSE). They are provided "AS IS", without warranties or conditions of any kind, and without any obligation to provide support. See the license for details.

The license covers this repository only. The CONNX JDBC server, its container image and its use are governed by your Software AG license agreement for CONNX. They are not covered by this license.
