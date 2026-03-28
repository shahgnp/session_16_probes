### Step 1: Create two pod with liveness and readiness probes

filename: `healthy-pod.yaml`
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: healthy-pod
  labels:
    app: demo-fleet
spec:
  containers:
  - name: web
    image: nginx:alpine
    # 1. Write custom text to index.html 2. Create health file 3. Start Nginx
    command: ["/bin/sh", "-c"]
    args:
      - "echo 'Response from POD 1 (Always Healthy)' > /usr/share/nginx/html/index.html && touch /tmp/readiness && exec nginx -g 'daemon off;'"
    ports:
    - containerPort: 80
    readinessProbe:
      exec:
        command: ["cat", "/tmp/readiness"] # Notice the space after colon!
      initialDelaySeconds: 2
      periodSeconds: 2
```

Apply
```bash
kubectl apply -f healthy-pod.yaml
```

filename: `unhealthy-pod.yaml`
```yaml
---
apiVersion: v1
kind: Pod
metadata:
  name: unhealthy-pod
  labels:
    app: demo-fleet
spec:
  containers:
  - name: web
    image: nginx:alpine
    command: ["/bin/sh", "-c"]
    args:
      - "echo 'Response from POD 2 (Will be broken soon)' > /usr/share/nginx/html/index.html && touch /tmp/readiness && exec nginx -g 'daemon off;'"
    ports:
    - containerPort: 80
    readinessProbe:
      exec:
        command: ["cat", "/tmp/readiness"]
      initialDelaySeconds: 2
      periodSeconds: 2
```

Apply
```bash
kubectl apply -f unhealthy-pod.yaml
```

### Step 2: Create a service

filename: `nginx-service.yaml`
```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: demo-service
spec:
  selector:
    app: demo-fleet
  ports:
  - port: 80
    targetPort: 80
```

Apply
```bash
kubectl apply -f nginx-service.yaml
```

### Step 3: Create a service monitor
Open a new terminal and execute

```bash
kubectl run monitor -i --tty --rm --image=curlimages/curl -- sh -c "while true; do curl -s http://nginx-service.<namespace>.svc.cluster.local; echo ''; sleep 1; done" # Replace with your namespace in the placeholder.
```

You should get
```
All commands and output from this session will be recorded in container logs, including credentials and sensitive information passed through the command prompt.
If you don't see a command prompt, try pressing enter.
Response from POD 1 (Always Healthy)

Response from POD 2 (Will be broken soon)

Response from POD 2 (Will be broken soon)

Response from POD 2 (Will be broken soon)

Response from POD 2 (Will be broken soon)

Response from POD 2 (Will be broken soon)

Response from POD 1 (Always Healthy)

Response from POD 1 (Always Healthy)
```

Observe that the traffic keeps switching between POD 1 and POD 2


### Step 4: Observe the service endpoints

```bash
kubectl get endpoints nginx-service
```

```
Warning: v1 Endpoints is deprecated in v1.33+; use discovery.k8s.io/v1 EndpointSlice
NAME            ENDPOINTS                                            AGE
kubernetes      10.0.60.223:6443,10.0.60.224:6443,10.0.60.225:6443   39d
nginx-service   10.42.4.236:80,10.42.5.254:80                        30s
php-apache      <none>                                               2d15h
```

```bash
kubectl get po healthy-pod unhealthy-pod -owide
```

```
NAME            READY   STATUS    RESTARTS   AGE     IP            NODE      NOMINATED NODE   READINESS GATES
healthy-pod     1/1     Running   0          3m16s   10.42.4.236   worker2   <none>           <none>
unhealthy-pod   1/1     Running   0          2m50s   10.42.5.254   worker3   <none>           <none>
```

Lookout for the clusterIP of pods in the endpoints of the service. This means that the service is routing traffic to it's listed endpoints i.e. the pods which the service selector has selected.

### Step 5: Remove /tmp/readiness in unhealthy pod i.e readiness probe failue

```bash
kubectl exec unhealthy-pod -- rm /tmp/readiness
```

### Step 6: Observe the pod and traffic monitor

```bash
kubectl get po
```

```
NAME            READY   STATUS    RESTARTS   AGE
healthy-pod     1/1     Running   0          7m31s
monitor         1/1     Running   0          3m7s
unhealthy-pod   0/1     Running   0          7m5s
```

unhealthy-pod is 0/1. Hence, the traffic is not routed through it. The traffic monitor terminal also verifies the same.

Observe the endpoint
```
kubectl get ep
```

Notice that the clusterIP of unhealthy pod is removed.

### Clean up

```bash
kubectl delete -f healthy-pod.yaml
kubectl delete -f unhealthy-pod.yaml
kubectl delete -f nginx-service.yaml
kubectl delete pod monitor
```
