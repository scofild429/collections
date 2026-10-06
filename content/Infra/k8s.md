---
title: "Kubernetes and Docker"
---

# Kubernetes and Docker

Practical notes, commands, and examples. Examples use the current namespace unless specified otherwise. Replace sample node names, image tags, paths, and domains with values for your environment. Image tags in examples are illustrative; pin tested versions or digests for reproducible deployments.

## Contents

- [Docker](#docker)
- [Kubernetes basics](#kubernetes-basics)
- [Workloads and rollouts](#workloads-and-rollouts)
- [Scheduling](#scheduling)
- [Resources and scaling](#resources-and-scaling)
- [Container patterns and lifecycle](#container-patterns-and-lifecycle)
- [Networking](#networking)
- [Storage and configuration](#storage-and-configuration)
- [Imperative kubectl commands](#imperative-kubectl-commands)
- [Ingress example](#ingress-example)
- [Milvus on a local cluster](#milvus-on-a-local-cluster)

## Docker

### Image names, builds, and pushes

An unqualified name such as `nginx` resolves to `docker.io/library/nginx:latest`. A name such as `my-user/my-app` uses Docker Hub but does not receive the `library/` namespace.

```bash
docker pull nginx:alpine
docker build -t my-app:dev .
docker build -t my-app:dev -f /path/to/Dockerfile /path/to/build-context

docker tag my-app:dev my-user/my-app:v1
docker login
docker push my-user/my-app:v1
```

`-t` assigns an image name and optional tag. Tagging creates another reference to the image; it does not rename or delete the original reference. `docker push` takes the destination image reference, not a source and destination pair.

Local sources for `COPY` and `ADD` normally come from the build context. `COPY --from` can read another build stage, image, or named context; `ADD` also supports remote sources. See the [Dockerfile reference](https://docs.docker.com/reference/dockerfile/).

### Dockerfile instructions

| Instruction | Purpose |
|---|---|
| `EXPOSE 5000` | Documents the container port; does not publish it on the host |
| `CMD echo "Hello World"` | Shell form; uses a shell and supports shell expansion |
| `CMD ["echo", "Hello World"]` | Exec form; runs the executable directly |
| `ENTRYPOINT ["executable"]` | Defines the executable; runtime arguments normally replace `CMD` arguments |

`docker run -p 80:5000 my-app:dev` publishes host port 80 to container port 5000 even without `EXPOSE`. The application must listen on an address reachable through the container network, usually `0.0.0.0`.

Arguments after the image override `CMD`. With an exec-form `ENTRYPOINT`, those arguments are passed to the entrypoint. Override the entrypoint with `docker run --entrypoint ...`; the spelling is **ENTRYPOINT**, not ENTRYPORT. Shell-form entrypoints have different argument and signal behavior. See [CMD and ENTRYPOINT](https://docs.docker.com/reference/dockerfile/#entrypoint).

### Multi-stage builds and microservices

Multi-stage builds separate build tools from the final runtime image. Copy only required artifacts into the final stage to reduce image size and unnecessary dependencies.

Microservices can support independent scaling, deployment, technology choices, and smaller codebases. Fault isolation depends on design: a failed dependency can still cause cascading failures. Distributed systems also introduce networking, observability, and operational complexity.

## Kubernetes basics

### Contexts and namespaces

A kubeconfig context groups a cluster reference, a user reference, and a default namespace. Changing its namespace does not change its cluster or credentials. Namespaces must exist before creating namespaced resources in them.

```bash
kubectl config current-context
kubectl create namespace app1-ns
kubectl config set-context --current --namespace=app1-ns
kubectl get pods -n app1-ns
kubectl get pods --all-namespaces
```

### Pod commands

A Pod is the smallest deployable unit. Its containers share networking and can share mounted volumes.

```bash
kubectl apply -f pod.yaml
kubectl get pods
kubectl get pods -o wide
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl logs <pod-name> -c <container-name>
kubectl logs -f <pod-name>
kubectl logs <pod-name> -c <container-name> --previous
kubectl exec -it <pod-name> -- /bin/sh
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
kubectl delete pod <pod-name>
```

The requested shell must exist in the image. Minimal images may have no shell. A workload controller may replace a deleted Pod.

### Labels and selectors

Labels are key-value metadata used to group and select Kubernetes objects. They can be attached to Nodes, Pods, Services, and other objects; they are not limited to Pods. Containers are not independently selectable API objects.

```bash
kubectl label node my-second-cluster-worker env=prod
kubectl label node my-second-cluster-worker storage=ssd
kubectl label node my-second-cluster-worker storage-
kubectl get pods -l 'app=nginx,environment notin (development)'
```

Services and ReplicationControllers use a simple equality-based selector map. ReplicaSets and Deployments use `matchLabels` and/or `matchExpressions`. Jobs normally generate selectors automatically; CronJobs create Jobs. Supported selector fields depend on the API resource.

Deployment selector fragment, placed under `spec`:

```yaml
selector:
  matchLabels:
    app: nginx
  matchExpressions:
    - key: environment
      operator: NotIn
      values: [development]
    - key: tier
      operator: Exists
    - key: debug
      operator: DoesNotExist
```

All requirements in this selector must match, and the Pod template labels must satisfy it. `In` and `NotIn` take values; `Exists` and `DoesNotExist` do not. `NotIn` also matches objects where the key is absent. See [labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/).

### Annotations

Annotations store descriptive or tool-specific metadata. Use `metadata.annotations` for the object itself and `spec.template.metadata.annotations` for Pods created by a workload. Changing a Deployment's Pod template triggers a rollout; changing only its top-level metadata does not.

## Workloads and rollouts

| Workload | Role |
|---|---|
| ReplicationController | Legacy controller maintaining a requested number of Pods |
| ReplicaSet | Maintains replicas and supports richer selectors; usually managed by a Deployment |
| Deployment | Manages ReplicaSets, rolling updates, and rollback of Pod templates |
| StatefulSet | Manages Pods with stable identities and storage associations |
| DaemonSet | Runs a Pod on each eligible node, often for networking, logging, or monitoring |
| Job | Runs a task to completion, with retry behavior |
| CronJob | Creates Jobs on a schedule |

A DaemonSet need not run on every node: scheduling constraints still apply. `kube-proxy` is commonly deployed as a DaemonSet, depending on the cluster networking implementation.

### Deployment update and rollback

A new Deployment revision is created when `spec.template` changes, such as an image update. Scaling or changing a top-level annotation alone does not create a revision. Rollback restores a retained Pod template, not external data or every Deployment field. See [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/).

The following is a lab workflow. Set `INITIAL_IMAGE` and `UPDATED_IMAGE` to two tested, distinct NGINX image tags or digests first.

```bash
: "${INITIAL_IMAGE:?Set an initial NGINX image reference}"
: "${UPDATED_IMAGE:?Set an updated NGINX image reference}"

kubectl create deployment mynginx --image="$INITIAL_IMAGE" --replicas=2
kubectl annotate deployment mynginx \
  kubernetes.io/change-cause="Initial release" --overwrite
kubectl rollout status deployment/mynginx
kubectl rollout history deployment/mynginx

# Pause to apply the new description and image before starting the rollout.
kubectl rollout pause deployment/mynginx
kubectl annotate deployment mynginx \
  kubernetes.io/change-cause="Update NGINX image" --overwrite
kubectl set image deployment/mynginx nginx="$UPDATED_IMAGE"
kubectl rollout resume deployment/mynginx
kubectl rollout status deployment/mynginx
kubectl rollout history deployment/mynginx
kubectl exec deployment/mynginx -- nginx -v

# Inspect retained revisions before choosing one.
kubectl rollout history deployment/mynginx --revision=1
kubectl rollout undo deployment/mynginx --to-revision=1
kubectl rollout status deployment/mynginx
kubectl exec deployment/mynginx -- nginx -v

# Clean up the lab deployment.
kubectl delete deployment mynginx
```

`kubernetes.io/change-cause` supplies the history description; a generic `description` annotation does not. Use explicit annotations instead of the obsolete `--record` workflow. `kubectl set image` supports several workload types, but Deployment revision management is distinct from updating a bare ReplicaSet or ReplicationController.

## Scheduling

### Choosing a mechanism

| Mechanism | What it controls |
|---|---|
| `nodeName` | Direct assignment to a node, bypassing the scheduler |
| `nodeSelector` | Hard requirement: all specified node labels must match |
| Required node affinity | Hard requirement with richer label expressions |
| Preferred node affinity | Weighted preferences among feasible nodes |
| Taints and tolerations | Whether a Pod can tolerate a node's scheduling or eviction restrictions |

A toleration permits placement; it does not attract a Pod to a node or guarantee placement. Combine it with node affinity when a workload must run on dedicated nodes.

### Manual assignment and static Pods

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-manual
spec:
  nodeName: my-second-cluster-worker2
  containers:
    - name: nginx
      image: nginx:alpine
```

Direct assignment bypasses scheduler checks, including `NoSchedule` taints. The kubelet can still reject a Pod, and an untolerated `NoExecute` taint can evict it.

Static Pods are managed directly by the kubelet from a configured source. The directory is commonly `/etc/kubernetes/manifests` on kubeadm clusters, but is configurable. The kubelet creates a mirror Pod in the API for visibility. Deleting the mirror does not stop the static Pod; change or remove its source manifest. See [static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/).

### nodeSelector

Label the intended node:

```bash
kubectl label node my-second-cluster-worker storage=ssd --overwrite
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: storage-aware-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: storage-aware-nginx
  template:
    metadata:
      labels:
        app: storage-aware-nginx
    spec:
      nodeSelector:
        storage: ssd
      containers:
        - name: nginx
          image: nginx:alpine
```

All `nodeSelector` entries must match the same node. Matching labels do not override resource constraints or taints.

### Node affinity

Pod-spec fragment, under `spec` for a Pod or `spec.template.spec` for a Deployment:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: storage
              operator: In
              values: [ssd, hdd]
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 10
        preference:
          matchExpressions:
            - key: storage
              operator: In
              values: [ssd]
      - weight: 5
        preference:
          matchExpressions:
            - key: storage
              operator: In
              values: [hdd]
```

Expressions within one term are **AND**; separate required `nodeSelectorTerms` are **OR**. If both `nodeSelector` and node affinity are specified, both must be satisfied. `IgnoredDuringExecution` means later label changes do not evict an already running Pod. Preferred rules contribute to scheduler scoring rather than guaranteeing a node choice.

| Operator | Match |
|---|---|
| `In` | Label value belongs to the set |
| `NotIn` | Label value is outside the set, or the key is absent |
| `Exists` | Label key exists |
| `DoesNotExist` | Label key is absent |
| `Gt`, `Lt` | Label value is an integer greater or less than the specified integer |

Zone selection uses the same mechanism with `topology.kubernetes.io/zone`. See [assigning Pods to nodes](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/).

### Taints and tolerations

Taints belong to Nodes; tolerations belong to Pods. Taints are separate from labels.

```bash
kubectl taint node my-second-cluster-worker storage=ssd:NoSchedule
kubectl taint node my-second-cluster-worker2 storage=hdd:NoSchedule
kubectl taint node my-second-cluster-worker storage=ssd:NoSchedule-
```

| Effect | Behavior for a Pod without a matching toleration |
|---|---|
| `NoSchedule` | Prevents new scheduling; does not evict existing Pods |
| `PreferNoSchedule` | Scheduler tries to avoid the node |
| `NoExecute` | Prevents scheduling and evicts existing Pods |

Pod-spec fragment:

```yaml
tolerations:
  - key: storage
    operator: Equal
    value: ssd
    effect: NoSchedule
```

`Equal` matches the key and value; `Exists` ignores the value. A specified effect must match; omitting the toleration's effect matches all effects for the matching key/value. Every relevant untolerated taint still matters.

`tolerationSeconds` applies only to `NoExecute`: it limits how long the Pod tolerates that taint before eviction. Omitting it tolerates that matching taint indefinitely, not all possible eviction causes. See [taints and tolerations](https://kubernetes.io/docs/concepts/scheduling-eviction/taint-and-toleration/).

## Resources and scaling

### Requests and limits

Resources are commonly configured per container. Pod-level CPU and memory resource settings are also available on supported Kubernetes versions and feature configurations; container-level settings are not the only option. See [Pod-level resources](https://kubernetes.io/docs/tasks/configure-pod-container/assign-pod-level-resources/).

Container fragment:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 256Mi
```

Requests inform scheduling; a container can use more than its request when capacity and limits permit. CPU limits are enforced through throttling. Memory limits are enforced reactively through OOM handling and can lead to `OOMKilled`; free memory elsewhere on the node does not make exceeding a container's limit safe. Node-pressure eviction is a separate mechanism. See [resource management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/).

### Monitoring and Metrics Server

Metrics Server supplies resource metrics for `kubectl top` and resource-based autoscaling. It is not a full monitoring or historical metrics system. Check its compatibility matrix and kubelet connectivity/TLS requirements before installation. See the [Metrics Server project](https://github.com/kubernetes-sigs/metrics-server).

```bash
# Convenience installation; pin a compatible release for repeatability.
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
kubectl top nodes
kubectl top pods
kubectl top pods --containers
```

### Scaling options

| Type | Manual change | Automation |
|---|---|---|
| Horizontal workload scaling | Change `spec.replicas` | Horizontal Pod Autoscaler (HPA) |
| Vertical workload scaling | Change resource requests/limits | Vertical Pod Autoscaler (VPA), installed separately |
| Node scaling | Add/remove nodes or change node capacity | Node autoscaling infrastructure |

HPA uses resource, custom, or external metrics. CPU utilization targets are relative to CPU requests, so relevant requests must be defined. VPA recommends or adjusts resource requests according to its configuration. Pod replacement versus in-place resizing depends on cluster support and update mode. See [workload autoscaling](https://kubernetes.io/docs/concepts/workloads/autoscaling/).

```bash
kubectl scale deployment mynginx --replicas=3
```

Changing `replicas` is horizontal scaling, not vertical scaling. Resizing a node may require replacement or restart, but can be used in production with appropriate capacity and disruption planning. A multi-node local cluster still shares the physical host's resources.

## Container patterns and lifecycle

### Init containers

Regular init containers run sequentially and must finish successfully before application containers start.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: init-demo
spec:
  initContainers:
    - name: check-api
      image: curlimages/curl:latest
      command:
        - sh
        - -c
        - |
          until curl -fsS --max-time 5 https://kubernetes.io/ >/dev/null; do
            echo 'Waiting for endpoint...'
            sleep 5
          done
  containers:
    - name: main-app
      image: nginx:alpine
```

This demonstrates waiting for an HTTPS endpoint, not checking the cluster's Kubernetes API. Replace it with the actual dependency check.

### Sidecars, ambassadors, and adapters

A sidecar runs alongside the main application. An ambassador handles communication on its behalf; an adapter transforms data or output into another format.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: sidecar-health-demo
spec:
  containers:
    - name: main-app
      image: nginx:alpine
      ports:
        - containerPort: 80
    - name: health-logger
      image: curlimages/curl:latest
      command:
        - sh
        - -c
        - |
          while true; do
            if curl -fsS --max-time 3 http://localhost:80/ >/dev/null; then
              echo 'Main app is reachable'
            else
              echo 'Main app is unreachable'
            fi
            sleep 5
          done
```

Containers share the Pod network, so `localhost` reaches the main container. This loop logs reachability; it does not replace readiness or liveness probes. Traditional sidecars use `containers`; native sidecars use `initContainers` with container-level `restartPolicy: Always` and have specialized startup/shutdown ordering. See [sidecar containers](https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/).

### Termination and restart policies

The default termination grace period is 30 seconds. The kubelet normally requests graceful shutdown, then forcibly stops remaining processes after the grace period. Lifecycle hooks consume time within that period.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-graceful
spec:
  terminationGracePeriodSeconds: 30
  containers:
    - name: nginx
      image: nginx:alpine
```

```bash
kubectl delete pod nginx-graceful
# Force deletion removes the API object without waiting for shutdown confirmation.
kubectl delete pod mypod --force --grace-period=0
```

Force deletion does not guarantee the process has stopped on its node.

| Pod restart policy | Container behavior |
|---|---|
| `Always` | Restart after any exit; default and required for ordinary Deployment Pods |
| `OnFailure` | Restart after a nonzero exit |
| `Never` | Do not restart the terminated container |

Jobs allow `OnFailure` or `Never`. With `Never`, the Job controller may still create replacement Pods. A standalone Pod can use any of the three. Native sidecars and newer per-container restart features are exceptions to a purely Pod-wide model. See [Pod lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/).

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: batch-job-example
spec:
  backoffLimit: 3
  template:
    spec:
      restartPolicy: OnFailure
      containers:
        - name: processor
          image: busybox:stable
          command: [sh, -c, "echo Processing complete"]
```

### Image pull policies

| Policy | Behavior |
|---|---|
| `Always` | Resolve the image reference at each start; cached content can still be reused |
| `IfNotPresent` | Pull only if the image is absent locally |
| `Never` | Use local content only; fail if unavailable |

When omitted, `:latest` or no tag defaults to `Always`; another tag or a digest defaults to `IfNotPresent`. The default is set at object creation and does not automatically change when the image reference changes. See [container images](https://kubernetes.io/docs/concepts/containers/images/).

### Pod phases and debugging

Pod phases are `Pending`, `Running`, `Succeeded`, `Failed`, and `Unknown`. `Running` does not imply readiness. `CrashLoopBackOff` and `ImagePullBackOff` are displayed waiting/backoff reasons; `OOMKilled` is a container termination reason, not a Pod phase.

Start with `kubectl describe pod`, container logs, and `kubectl logs --previous`. The [Pod lifecycle reference](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/) describes the distinction between Pod phase and container state.

### Health probes

The kubelet performs probes for individual containers.

| Probe | Failure response |
|---|---|
| Readiness | Marks the container unready; the Pod normally stops receiving Service traffic |
| Liveness | Triggers container termination and restart according to its restart policy |
| Startup | Gates readiness/liveness until startup succeeds; repeated failure triggers termination |

Probe handlers include HTTP, TCP, exec commands, and gRPC. Readiness failure does not itself restart a container. See [probe configuration](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/).

Container fragment for NGINX:

```yaml
ports:
  - name: http
    containerPort: 80
startupProbe:
  httpGet:
    path: /
    port: http
  periodSeconds: 5
  failureThreshold: 30
readinessProbe:
  httpGet:
    path: /
    port: http
  periodSeconds: 5
livenessProbe:
  httpGet:
    path: /
    port: http
  periodSeconds: 10
```

## Networking

### Services

A Service provides a stable abstraction over endpoints, often selected Pods. Most Services have a stable virtual IP and DNS name. Headless Services (`clusterIP: None`) and `ExternalName` are exceptions. Traffic distribution depends on the implementation and policies; it is not necessarily perfectly even. See [Services](https://kubernetes.io/docs/concepts/services-networking/service/).

| Type | Purpose |
|---|---|
| `ClusterIP` | Default; exposes a cluster-internal virtual IP |
| `NodePort` | Exposes a port on nodes, usually in the range 30000–32767 |
| `LoadBalancer` | Requests a load balancer from a supporting implementation |
| `ExternalName` | Returns a DNS alias; does not proxy traffic or allocate a virtual IP |

A Service's `port` is its listening port; `targetPort` identifies the backend port. A container's `containerPort` documents a port and can name it; it does not start a listener or publish a host port. These numbers do not all need to match.

### NetworkPolicy

A NetworkPolicy selects Pods in its namespace and defines allowed ingress and/or egress. Enforcement requires a compatible network plugin. Without policies isolating a Pod in a given direction, traffic in that direction is allowed by default; applicable allow rules are additive. See [NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/).

Example: deny inbound traffic to all Pods in the current namespace unless another policy allows it.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
    - Ingress
```

## Storage and configuration

### emptyDir

`emptyDir` provides temporary Pod-scoped storage. It survives container restarts but is deleted when the Pod is removed from its node. Containers mounting the same volume see the same files. A memory-backed `emptyDir` (`medium: Memory`) consumes memory. See [volumes](https://kubernetes.io/docs/concepts/storage/volumes/#emptydir).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-example
spec:
  containers:
    - name: writer
      image: busybox:stable
      command: [sh, -c, "echo shared-data > /data/message; sleep 3600"]
      volumeMounts:
        - name: temp-storage
          mountPath: /data
    - name: reader
      image: busybox:stable
      command: [sh, -c, "sleep 3600"]
      volumeMounts:
        - name: temp-storage
          mountPath: /data
  volumes:
    - name: temp-storage
      emptyDir: {}
```

### Downward API

The Downward API exposes selected Pod metadata and resource fields as environment variables or mounted files. It is not a channel for collecting another container's live application metrics. See [Downward API](https://kubernetes.io/docs/concepts/workloads/pods/downward-api/).

Container fragment:

```yaml
env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_NAMESPACE
    valueFrom:
      fieldRef:
        fieldPath: metadata.namespace
```

### PersistentVolumes, claims, and StorageClasses

| Object | Scope | Purpose |
|---|---|---|
| PersistentVolume (PV) | Cluster | Represents provisioned storage and its capabilities |
| PersistentVolumeClaim (PVC) | Namespace | Requests capacity and access characteristics; binds to a PV |
| StorageClass | Cluster | Defines provisioning behavior and parameters |
| Pod volume | Pod | References a PVC or another volume source for containers to mount |

A StorageClass does not replace PVs: dynamic provisioning creates PVs for PVCs. Binding considers storage class, capacity, access modes, volume mode, and other constraints. `volumeName` requests a particular PV but does not bypass compatibility checks.

Key fields are `spec.capacity.storage` and `spec.persistentVolumeReclaimPolicy` on a PV, and `spec.resources.requests.storage` on a PVC. `Retain` preserves storage for manual reclamation; `Delete` requests deletion of the volume and backing storage when supported by the provisioner. Protection finalizers defer deletion of in-use PVCs or bound PVs. Stop consumers before deleting their storage. See [persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

### hostPath lab example

`hostPath` mounts a path from a particular node. Pods on different nodes do not share its data. It exposes host files and is appropriate here only as a local lab example. In production, use storage appropriate to your availability and security requirements.

This example explicitly places the consumer on the node holding the data; replace the hostname in both places with an actual node label value. `manual` is an arbitrary matching class name for this statically provisioned example.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: log-volume
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /opt/volume/nginx
    type: DirectoryOrCreate
  nodeAffinity:
    required:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values: [my-second-cluster-worker]
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: manual
  volumeName: log-volume
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: volume-demo
spec:
  nodeSelector:
    kubernetes.io/hostname: my-second-cluster-worker
  containers:
    - name: nginx
      image: nginx:alpine
      volumeMounts:
        - name: web-data
          mountPath: /usr/share/nginx/html
  volumes:
    - name: web-data
      persistentVolumeClaim:
        claimName: my-pvc
```

The directory must contain an `index.html` if you want NGINX to serve a page. `ReadWriteOnce` means writable from one node, not necessarily from only one Pod, and the capacity declaration does not impose a host-directory disk quota.

To mount a host path directly without a PVC, replace the Pod's volume source with this fragment:

```yaml
volumes:
  - name: web-data
    hostPath:
      path: /opt/volume/nginx
      type: DirectoryOrCreate
```

The correct keys are `mountPath` and `claimName`, not `path` under `volumeMounts` or `chainName` under `persistentVolumeClaim`.

### ConfigMaps

ConfigMaps hold non-confidential configuration. This example injects two environment variables and mounts one HTML file through a directory volume.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: frontend-cm
data:
  APP: frontend
  ENVIRONMENT: production
  index.html: |
    <!DOCTYPE html>
    <html>
      <head><title>Welcome</title></head>
      <body><h1>Hello from Kubernetes!</h1></body>
    </html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend-container
          image: nginx:alpine
          env:
            - name: APP
              valueFrom:
                configMapKeyRef:
                  name: frontend-cm
                  key: APP
            - name: ENVIRONMENT
              valueFrom:
                configMapKeyRef:
                  name: frontend-cm
                  key: ENVIRONMENT
          volumeMounts:
            - name: html-volume
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: html-volume
          configMap:
            name: frontend-cm
            items:
              - key: index.html
                path: index.html
```

The directory mount hides pre-existing files in the image at that directory. To mount just one file while retaining the other files, replace the container's `volumeMounts` with:

```yaml
volumeMounts:
  - name: html-volume
    mountPath: /usr/share/nginx/html/index.html
    subPath: index.html
    readOnly: true
```

### Secrets and configuration updates

Secrets hold confidential data and can also be consumed through environment variables or volumes. They are distinct API objects: use `secretKeyRef` or a `secret` volume source. Base64 encoding is not encryption; configure suitable access control and encryption at rest. See [Secrets](https://kubernetes.io/docs/concepts/configuration/secret/).

| Consumption method | Behavior after ConfigMap/Secret changes |
|---|---|
| Environment variables | Existing process values do not change; replace the Pod to pick up updates |
| Normal volume projection | Updates propagate eventually; the application must reread or reload files |
| `subPath` mount | Does not receive automatic projected updates |

See [ConfigMap consumption and updates](https://kubernetes.io/docs/concepts/configuration/configmap/).

## Imperative kubectl commands

`--dry-run=client -o yaml` generates a manifest without creating the object; it is not full server-side validation. `--dry-run=server` submits a dry-run request for API validation and admission processing without persistence.

```bash
kubectl create deployment backend-deploy \
  --image=hashicorp/http-echo \
  --replicas=3 \
  --port=5678 \
  --dry-run=client -o yaml > backend-deploy.yaml

# Review and apply the generated Deployment before exposing it.
kubectl apply -f backend-deploy.yaml
kubectl expose deployment backend-deploy \
  --type=ClusterIP \
  --port=9090 \
  --target-port=5678 \
  --name=backend-svc \
  --dry-run=client -o yaml > backend-svc.yaml
kubectl apply --dry-run=server -f backend-svc.yaml
kubectl apply -f backend-svc.yaml
```

`kubectl expose deployment` reads an existing Deployment even with client dry-run. For this example, traffic to Service port 9090 reaches the application's listener on port 5678.

`kubectl run --port` and `kubectl create deployment --port` declare a container port. `kubectl expose --port` declares the Service port. Specify `--target-port` explicitly when ports differ; its default is the Service port, not necessarily the declared container port. See [`kubectl expose`](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_expose/).

`kubectl expose` supports `--type=NodePort` but has no flag to choose the specific node port. Set `spec.ports[].nodePort` in a manifest, or generate a Service using:

```bash
kubectl create service nodeport backend-nodeport \
  --tcp=9090:5678 --node-port=30090 \
  --dry-run=client -o yaml
```

Review the generated selector and change it to `app: backend-deploy` if it should target the Deployment above.

## Ingress example

An Ingress defines HTTP/HTTPS routing rules; a compatible controller must implement them. The original notes manually installed community Ingress NGINX `v1.10.0` with incomplete controller setup. That controller project retired in March 2026. Use a maintained controller and its supported installation process; the Ingress API itself is distinct from that retired implementation. See the [Kubernetes retirement statement](https://kubernetes.io/blog/2026/01/29/ingress-nginx-statement/).

Prerequisites for this application example:

1. Install a maintained Ingress controller and expose it so clients can reach it.
2. Find its IngressClass with `kubectl get ingressclass`; replace `example-class` below.
3. Create the `ingress-demo` namespace and a TLS Secret containing a certificate for `webapp.example.test`.
4. Resolve that hostname to the controller's reachable address, through DNS or a local hosts entry.

```bash
kubectl create namespace ingress-demo
kubectl -n ingress-demo create secret tls webapp-tls-secret \
  --cert=/path/to/tls.crt --key=/path/to/tls.key
```

Save the following as `ingress-demo.yaml` and apply it after replacing the class. Controller installation, RBAC, and controller-specific configuration belong to the selected controller's supported deployment package.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-html
  namespace: ingress-demo
data:
  index.html: |
    <html><body><h1>Hello from WebApp!</h1></body></html>
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
  namespace: ingress-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
        - name: nginx
          image: nginx:alpine
          ports:
            - name: http
              containerPort: 80
          volumeMounts:
            - name: html
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: html
          configMap:
            name: webapp-html
---
apiVersion: v1
kind: Service
metadata:
  name: webapp-service
  namespace: ingress-demo
spec:
  selector:
    app: webapp
  ports:
    - port: 80
      targetPort: http
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: ingress-demo
spec:
  ingressClassName: example-class
  tls:
    - hosts:
        - webapp.example.test
      secretName: webapp-tls-secret
  rules:
    - host: webapp.example.test
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: webapp-service
                port:
                  number: 80
```

```bash
kubectl apply -f ingress-demo.yaml
kubectl -n ingress-demo rollout status deployment/webapp
kubectl -n ingress-demo get ingress,service,pods
curl --cacert /path/to/ca.crt https://webapp.example.test/
```

The TLS Secret and backend Service must be in the Ingress namespace. A root path does not require a controller-specific rewrite annotation. See [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/).

## Milvus on a local cluster

### Create a kind cluster

Save this as `kind-milvus.yaml`. It is a kind configuration file, not a manifest for `kubectl apply`.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
  - role: worker
  - role: worker
  - role: worker
```

```bash
kind create cluster --name milvus --config kind-milvus.yaml
kubectl get nodes
kubectl get storageclass
```

Five worker containers do not provide five machines' worth of capacity or failure isolation. Ensure enough host resources and a suitable default StorageClass for the chart's PVCs.

### Inspect and pin the Helm chart

Milvus dependencies and Helm values vary by chart version. The original Kafka/Pulsar overrides should not be copied across releases unchecked: current documentation uses Woodpecker as the default message queue. Follow the [Milvus Helm installation guide](https://milvus.io/docs/install_cluster-helm.md) for the selected release.

```bash
helm repo add milvus https://zilliztech.github.io/milvus-helm/
helm repo update
helm search repo milvus/milvus --versions

# Set CHART_VERSION to a reviewed version from the preceding output.
: "${CHART_VERSION:?Set a Milvus Helm chart version}"
helm show values milvus/milvus --version "$CHART_VERSION" > milvus-values.yaml
```

Review `milvus-values.yaml`, including cluster mode, resource requirements, persistence, etcd replicas, object storage, and the message queue. If choosing Kafka, check the selected chart's exact values and the Milvus configuration for that queue; a historical `kafka.replicaCount` override may be ignored by newer charts.

Render before installing:

```bash
helm template my-milvus milvus/milvus \
  --version "$CHART_VERSION" \
  --namespace milvus \
  -f milvus-values.yaml > milvus-rendered.yaml

helm install my-milvus milvus/milvus \
  --version "$CHART_VERSION" \
  --namespace milvus --create-namespace \
  -f milvus-values.yaml

kubectl -n milvus get pods,pvc
helm status my-milvus -n milvus
```

## Review notes

The conversion consolidates repeated selector, affinity, and rollout sections; removes stale captured Pod names and command output; fixes malformed YAML and command flags; and replaces the obsolete Ingress controller installation with an application example that states its prerequisites. The original source's Org export settings were omitted because they do not apply to Markdown.

The examples are study material, not a tested deployment bundle. Version-sensitive behavior should be checked against the documentation for your cluster and selected chart versions.
