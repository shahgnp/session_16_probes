
## Probes in action: What happens when probes don't succeed

### Step 1: Create a pod with liveness and readiness probes and it's satisfying condition

filename: `probe-demo.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
  labels:
    app: probe-demo
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    # Create a file /tmp/healthy on startup
    command: ["/bin/sh", "-c", "touch /tmp/liveness /tmp/readiness; sleep 3600"]
    
    # LIVENESS: If /tmp/liveness is gone, KILL ME -> Restart Pod
    livenessProbe:
      exec:
        command:
        - cat
        - /tmp/liveness
      initialDelaySeconds: 5
      periodSeconds: 5

    # READINESS: If /tmp/readiness is gone, STOP TRAFFIC -> 0/1 NotReady
    readinessProbe:
      exec:
        command:
        - cat
        - /tmp/readiness
      initialDelaySeconds: 5
      periodSeconds: 5
```

### Step 2: Wait and Obeserve the pod

```bash
kubectl get po
```

```
NAME         READY   STATUS    RESTARTS   AGE
probe-demo   1/1     Running   0          26s
```
1/1 means that the pod is ready to serve traffic i.e readiness probe is successfuly
Running and Restart = 0 means the pod is alive i.e liveness probe is successful

### Step 3: Delete the /tmp/readiness file

```bash
kubectl exec -it probe-demo -- rm /tmp/readiness
```

### Step 4: Obeserve the readiness probe failure

```bash
kubectl get po probe-demo
```

```
NAME         READY   STATUS    RESTARTS   AGE
probe-demo   0/1     Running   0          26s
```
0/1 means the pod is not ready to serve traffic

Also,

```bash
kubectl describe po probe-demo
```

Look out for
```
Warning  Unhealthy  53s (x25 over 2m38s)  kubelet            Readiness probe failed: cat: can't open '/tmp/readiness': No such file or directory
```

### Step 5: Remove Delete the /tmp/liveness file

```bash
kubectl exec -it probe-demo -- rm /tmp/readiness
```

### Step 6: Observe the liveness probe failure

```bash
kubectl get po probe-demo
```

```
NAME         READY   STATUS    RESTARTS      AGE
probe-demo   1/1     Running   1 (47s ago)   5m27s
```

Notice that there is RESTARTS=1, which means the liveness probe failed and has restarted the pod.

But, why is the READY 1/1 despite deleting the /tmp/readiness file in earlier step? Also, why did the pod just restart only once and not repeatedly?

Because when the pod restarts it recreates the `/trmp/readiness` and `/tmp/liveness` files on it's own. Read line 12 of `probe-demo.yaml`.

### Step 7: Remove `touch /tmp/liveness /tmp/readiness` from the manifest and try again

```bash
kubectl delete -f probe-demo.yaml
```

Edit `probe-demo.yaml` and replace the command line with the below line
```yaml
command: ["/bin/sh", "-c", "sleep 3600"]
```

### Step 8: Wait and observe the probes

```bash
kubectl describe po probe-demo
```

Look out for which tells, both liveness and readiness probes have failed.
```
  Warning  Unhealthy  3s               kubelet            Liveness probe failed: cat: can't open '/tmp/liveness': No such file or directory
  Warning  Unhealthy  1s (x2 over 6s)  kubelet            Readiness probe failed: cat: can't open '/tmp/readiness': No such file or directory
```

Wait a few minutes and run
```bash
kubectl get po
```

```
NAME         READY   STATUS             RESTARTS      AGE
probe-demo   0/1     CrashLoopBackOff   5 (65s ago)   5m40s
```

RESTARTS count denotes that the pod has restarted itself many times and the STATUS is CrashLookBackOff which means the pod also means that the pod has restarted itself many times.

### Step 9: Clean up

```bash
kubectl delete -f probe-demo.yaml
```