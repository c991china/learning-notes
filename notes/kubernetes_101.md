# Kubernetes 101

kubectl 1.29 against a managed cluster (EKS/GKE-ish). I use k8s, I don't run the
control plane. Notes are from the app-developer side.

## The mental model

- **Pod** — one or more containers sharing a network namespace and volumes.
  Ephemeral; pods die and get recreated with new names.
- **Deployment** — manages ReplicaSets, which manage Pods. You rarely create
  Pods directly.
- **Service** — a stable virtual IP + DNS name in front of a set of pods,
  selected by labels.
- **Ingress** — HTTP routing from outside the cluster to Services.
- **ConfigMap / Secret** — config and credentials, mounted or injected as env.

The thing that clicked for me: labels + selectors wire everything together. A
Service finds its pods by label, not by name.

## Everyday commands

```bash
kubectl get pods -n myapp
kubectl get pods -o wide                 # + node and pod IP
kubectl get deploy,svc,ingress -n myapp
kubectl describe pod myapp-7d9f-abc      # events at the bottom. read those.
kubectl logs -f myapp-7d9f-abc -c app
kubectl exec -it myapp-7d9f-abc -- sh
kubectl port-forward svc/myapp 8080:80   # localhost:8080 -> service
```

`kubectl describe` is the first command when something is wrong. The Events
section at the bottom usually tells you exactly why.

## CrashLoopBackOff debugging

The pod restarts, backs off, restarts. Order I go in:

```bash
kubectl get pod <pod>                       # STATUS, RESTARTS
kubectl describe pod <pod>                  # Last State: Terminated, Reason, Exit Code
kubectl logs <pod> --previous               # logs from the crashed container!
```

> **gotcha**: `kubectl logs <pod>` shows the CURRENT container. After a crash
> the new container may have no logs yet. You need `--previous` (or `-p`) to see
> why the last one died. This one cost me a while.

Common exit codes:
- **0** but still restarting -> liveness probe failing (see below), or the
  process exits immediately and the container's job is to keep running.
- **1** -> app error. Read `--previous` logs.
- **137** -> SIGKILL. Usually OOM (`kubectl describe` shows `Reason: OOMKilled`)
  or the liveness probe killed it.
- **143** -> SIGTERM. Graceful shutdown or a probe.

## Probes: where I've shot myself in the foot

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: 8080 }
  initialDelaySeconds: 10
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /ready, port: 8080 }
  initialDelaySeconds: 5
```

- **readiness** failing -> pod removed from Service endpoints. Traffic stops.
  Safe-ish.
- **liveness** failing -> kubelet KILLS and restarts the container.

> **gotcha**: if liveness `initialDelaySeconds` is too short for a slow-starting
> app, the kubelet kills it before it ever finishes booting, forever. Symptom:
> constant restarts, logs show the app starting normally each time. Make the
> liveness initial delay generous and let readiness gate traffic.

## Resources: requests vs limits

```yaml
resources:
  requests: { cpu: 100m, memory: 128Mi }
  limits:   { cpu: 500m, memory: 256Mi }
```

- **requests** = what the scheduler reserves. Affects which node it lands on.
- **limits** = hard cap. CPU over limit = throttled. Memory over limit = OOMKilled.

> **gotcha**: a memory limit lower than the app's steady-state RSS means it gets
> OOMKilled under load, and the logs just stop. Check `kubectl top pod` and set
> the limit above peak, not above average.

## Services

```yaml
apiVersion: v1
kind: Service
metadata: { name: myapp }
spec:
  selector: { app: myapp }     # must match pod labels exactly
  ports:
    - port: 80
      targetPort: 8080         # the container's port
```

> **gotcha**: `port` is the Service port; `targetPort` is the container port.
> Mismatch here = connection refused from inside the cluster, even though the
> pod is Running and Ready. Also: an empty `Endpoints` list means the selector
> matched no pods.

```bash
kubectl get endpoints myapp     # if empty, selector is wrong
```

## Namespaces and context

```bash
kubectl config get-contexts
kubectl config use-context prod-cluster
kubectl config set-context --current --namespace=myapp   # stop typing -n
```

> **gotcha**: running a destructive command against the wrong context is the
> classic k8s disaster. I set a shell prompt that shows the current context
> (`kubectl config current-context`) so I can see prod before I `delete`.

## Notes

- `kubectl apply -f` is declarative and idempotent. `kubectl create` fails if it
  already exists. Prefer `apply`.
- `kubectl rollout status deploy/myapp` blocks until the rollout finishes or
  fails; `kubectl rollout undo deploy/myapp` rolls back. Use both in CI.
- ConfigMap changes do NOT auto-restart pods that mount them as env vars. You
  must restart the deployment (or use a checksum annotation to trigger it).
- `kubectl delete pod <pod>` on a Deployment-managed pod just recreates it. To
  actually remove, scale the deployment or delete the deployment.
