## Exploring Different types of probes in kubernetes: TCP and HTTP

### Step 1: Create a pod with tcpSocket type liveness and readiness

filename: `tcp-probe-demo.yaml`
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: tcp-probe-demo
spec:
  containers:
  - name: goproxy
    image: registry.k8s.io/goproxy:0.1
    ports:
    - containerPort: 8080
    readinessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 8080
      initialDelaySeconds: 15
      periodSeconds: 10
```

### Step 2: Obeserve the liveness and readiness probe

```bash
kubectl describe po tcp-probe-demo
```

Lookout for liveness and readiness field
```
Liveness:       tcp-socket :8080 delay=15s timeout=1s period=10s #success=1 #failure=3
Readiness:      tcp-socket :8080 delay=15s timeout=1s period=10s #success=1 #failure=3
```

### Step 3: Clean up

```bash
kubectl delete po tcp-probe-demo
```

### Step 4: Create a pod with httpGet type liveness

filename: `http-probe-demo.yaml`
```
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: liveness
  name: http-probe-demo
spec:
  containers:
  - name: liveness
    image: registry.k8s.io/e2e-test-images/agnhost:2.40
    args:
    - liveness
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
        httpHeaders:
        - name: Custom-Header
          value: Awesome
      initialDelaySeconds: 3
      periodSeconds: 3
```

### Step 5: Observe the probe

```bash
kubectl describe pod/http-probe-demo
```

Look out for
```
Liveness:       http-get http://:8080/healthz delay=3s timeout=1s period=3s #success=1 #failure=3
```

### Step 6: Clean up

```bash
kubectl delete po http-probe-demo
```