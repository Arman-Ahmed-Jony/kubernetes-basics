# Cha-Wala — answers

Cluster: Minikube v1.39.0, Kubernetes v1.37.0, Docker driver. Namespace for every app object: `cha-express`.

## Task 1 — Cluster, namespace, control plane

```text
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP    OS-IMAGE                         CONTAINER-RUNTIME
minikube   Ready    control-plane   9d    v1.37.0   192.168.49.2   Debian GNU/Linux 12 (bookworm)   containerd://2.3.4
```

Control plane components running on this node:

| Component | Where it runs |
| --- | --- |
| kube-apiserver | static pod `kube-apiserver-minikube` |
| etcd | static pod `etcd-minikube` |
| kube-scheduler | static pod `kube-scheduler-minikube` |
| kube-controller-manager | static pod `kube-controller-manager-minikube` |
| kubelet | process on the node, not a pod |

`kubelet` was confirmed on the node:

```text
556 /var/lib/minikube/binaries/v1.37.0/kubelet --bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.conf --config=/var/lib/kubelet/config.yaml --hostname-override=minikube --kubeconfig=/etc/kubernetes/kubelet.conf --node-ip=192.168.49.2
```

Also running in `kube-system`, and not part of the control plane: `kube-proxy`, `coredns`, `kindnet` (CNI), and `storage-provisioner` (Minikube addon).

## Task 3 — Are Kubernetes Secrets encrypted?

No. On this cluster a Secret is stored as base64. Base64 is an encoding, so anyone who can read the Secret object can recover the value.

The API object returns:

```text
ADMIN_PASSWORD (data): bWFtYS1jaGEtYWRtaW4tMjAyNg==
PAYMENT_API_KEY (data): cGtfdGVzdF9yYWhpbV9tYW1hX2R1bW15
```

Decoding the password:

```bash
kubectl get secret cha-secret -n cha-express -o jsonpath='{.data.ADMIN_PASSWORD}' | base64 -d
```

```text
mama-cha-admin-2026
```

Encryption at rest is also off. The `kube-apiserver` command has no `--encryption-provider-config` flag. Its arguments are the usual Minikube TLS, etcd, and admission flags only. Without that flag, etcd stores Secret data in the clear aside from the base64 encoding.

## Task 4 — Deployment and Service

`cha-web` runs `nginx:1.25-alpine` in the manifest (3 replicas), then Task 6 moves the running pods to `nginx:1.27-alpine`. The container receives `SHOP_NAME`, `CITY`, `THEME_COLOR`, and `OPENING_HOURS` from ConfigMap `cha-config`, and only `ADMIN_PASSWORD` from Secret `cha-secret`. `index.html` is a volume at `/usr/share/nginx/html`, not an environment variable. The Service is a NodePort on **30080**.

From a running pod before the bonus restart (`printenv` of `PAYMENT_API_KEY` exits non-zero, which is the check that it is absent):

```text
SHOP_NAME=Rahim Mama's Cha
CITY=Dhaka
THEME_COLOR=#C4622D
OPENING_HOURS=6:00 AM - 11:00 PM
ADMIN_PASSWORD=mama-cha-admin-2026
PAYMENT_API_KEY is not set
```

After the bonus restart the same five variables are present, with `THEME_COLOR=#1F6B4A`, and `PAYMENT_API_KEY` is still absent.

The menu was opened in a browser at `http://127.0.0.1:18080` (port-forward to the Service) and at NodePort 30080 from the Minikube node (`HTTP 200`). Screenshot: [menu.png](menu.png).

## Task 5 — GitHub Copilot

Prompt given to Copilot:

> Write a Kubernetes Deployment and NodePort Service in namespace cha-express. Deployment name cha-web, 3 replicas, image nginx:1.25-alpine, container name web. Env SHOP_NAME, CITY, THEME_COLOR, OPENING_HOURS from ConfigMap cha-config. Env ADMIN_PASSWORD only from Secret cha-secret. Mount ConfigMap key index.html at /usr/share/nginx/html. Set cpu/memory requests and limits. Service nodePort 30080, port 80, targetPort 80, selector app: cha-web. RollingUpdate maxUnavailable 0, maxSurge 1. Readiness probe HTTP GET / on port 80.

GitHub Copilot is not installed here (`gh` and the `copilot` CLI are not on `PATH`). No Copilot reply was written in its place. Paste the raw YAML under this heading and the correction list can be filled in from the real output.

### Raw Copilot output

(paste here)

### What had to change, and why

(fill in from the pasted output)

The hand-written manifests in `manifests/deployment.yaml` and `manifests/service.yaml` are the version that meets the assignment. When comparing Copilot's YAML to those files, check these points, which are the usual misses:

- Namespace `cha-express` on both objects.
- Image exactly `nginx:1.25-alpine`, container name `web`, 3 replicas.
- Four ConfigMap keys as individual env vars. `envFrom` would also inject `index.html`.
- Only `ADMIN_PASSWORD` from the Secret. `envFrom` on the Secret would also inject `PAYMENT_API_KEY`.
- `index.html` mounted on the directory `/usr/share/nginx/html`, not with `subPath` (a `subPath` mount would not pick up the bonus file change).
- Requests and limits both set.
- Service `nodePort: 30080`, `port: 80`, `targetPort: 80`.

## Task 6 — Rolling update and rollback

`kubernetes.io/change-cause` is set on the Deployment before each change. The controller copies that annotation onto the current ReplicaSet, so an older ReplicaSet was annotated directly when a later update would have overwritten its reason.

History after the update, the failed tag, the undo, and the bonus restart:

```text
REVISION  CHANGE-CAUSE
1         Initial release: nginx 1.25-alpine
3         Intentional bad image nginx:1.27-alpine-does-not-exist
4         Rollback to nginx 1.27-alpine (first rolled out as: Update image to nginx:1.27-alpine)
5         Restart pods to load updated THEME_COLOR
```

Revision 2 is absent because `kubectl rollout undo` reuses the previous ReplicaSet and assigns it a new revision number. That ReplicaSet is revision 4. It was created for the update to `nginx:1.27-alpine`, then restored by the undo. The cause text records both.

While the update was in progress, one new pod was `ContainerCreating` and the three pods on `nginx:1.25-alpine` stayed `Running`. Every sample of `http://127.0.0.1:30080` from the node returned `200`. Raw output: [evidence/rollout-update.txt](evidence/rollout-update.txt).

The bad tag `nginx:1.27-alpine-does-not-exist` produced `ImagePullBackOff`:

```text
Failed to pull image "nginx:1.27-alpine-does-not-exist": ... not found
Error: ImagePullBackOff
```

`kubectl rollout undo deployment/cha-web -n cha-express` restored `nginx:1.27-alpine`. The three ready pods are on that image. NodePort still returned `200`.

### Did the website go down?

No. The Deployment uses `maxUnavailable: 0` and `maxSurge: 1`, and the container has an HTTP readiness probe on `/`. During the good update, a new pod is added before any old pod is removed, and it joins the Service only after Nginx answers. During the bad update, the new pod never becomes Ready, so the old pods stay in the Service. A gap would need `maxUnavailable` above 0, or no readiness check, so an unready pod could replace a working one.

## Task 7 — Four bugs

`debug/cha-broken.yaml` was applied as given. Each bug was identified, then fixed, and `debug/cha-broken-fixed.yaml` was applied at the end. `kubectl diff -f debug/cha-broken-fixed.yaml` is empty.

The fixed Deployment still does not mount `index.html`. The original challenge only injects the ConfigMap with `envFrom`, so Nginx serves its built-in welcome page. That page returning through the Service is the check that port 80 works.

### 1. Image tag `nginx:1.27-alpinee`

- Symptom: both pods stuck in `ErrImagePull` / `ImagePullBackOff`, `0/1 Ready`.
- How: `kubectl describe pod` events.
- Cause: the tag has an extra `e`. The registry has no such image (`docker.io/library/nginx:1.27-alpinee: not found`).
- Fix: set the image to `nginx:1.27-alpine`.

This showed up first. The kubelet pulls the image before it configures environment variables.

### 2. ConfigMap name `cha-confg`

- Symptom: after the image fix, the new pod waited in `CreateContainerConfigError`.
- How: describe events: `Error: configmap "cha-confg" not found`.
- Cause: the name is missing the `i`. The real ConfigMap is `cha-config`.
- Fix: point `configMapRef.name` at `cha-config`. The pods then became `Running`.

### 3. Service selector `app: cha-brokn`

- Symptom: pods were `Running` with label `app=cha-broken`, and `wget http://cha-broken-svc/` returned `Connection refused`. The EndpointSlice had no addresses.
- How: compared `svc.spec.selector` (`app=cha-brokn`) with `kubectl get pods --show-labels`.
- Cause: the selector misses the `e`, so it matches no pods. A Service with no endpoints refuses the connection.
- Fix: selector `app: cha-broken`. The EndpointSlice then listed both pod IPs. Its port was still `8080`.

### 4. `targetPort: 8080`

- Symptom: endpoints existed, and `wget` to the Service still returned `Connection refused`.
- How: from `cha-web`, `wget` to the broken pod IP on port 80 returned `HTTP/1.1 200` (`Server: nginx/1.27.5`). The same request to port 8080 returned `Connection refused`. The EndpointSlice port was `8080`.
- Cause: this Nginx image listens on 80. The Service was sending traffic to 8080, where nothing is listening.
- Fix: `targetPort: 80`. `wget http://cha-broken-svc/` then returned the Nginx welcome page, and the EndpointSlice port is `80`.

## Bonus — ConfigMap update

Started from `THEME_COLOR=#C4622D` and an `index.html` whose CSS color was `#C4622D` and whose tagline did not say "Evening special".

Changed the ConfigMap to `THEME_COLOR=#1F6B4A` and edited `index.html` (CSS color `#1F6B4A`, tagline "Evening special..."), then applied it. No pod was restarted yet.

Immediately after apply, inside an already-running pod:

```text
env=#C4622D
file_color=#C4622D
file_tag=no evening special
```

About a minute later, same pod, still no restart:

```text
env=#C4622D
file_color=#1F6B4A
file_tag=Evening special
```

The file change was on the page (green header, "Evening special"). The environment variable was still the old value.

They differ because of how each one is consumed:

- `index.html` is a projected ConfigMap volume on `/usr/share/nginx/html` (a directory mount, not `subPath`). The kubelet refreshes that volume while the pod keeps running.
- `THEME_COLOR` is an environment variable. The container process receives it at start, and the process keeps that value.

`kubectl rollout restart deployment/cha-web -n cha-express` recreates the pods. After that, `THEME_COLOR=#1F6B4A` and the file still contains "Evening special". That restart is revision 5 in the history above.
