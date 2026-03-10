# Kubernetes Session 2: From Concepts to Practice
*Building on Session 1: Now Let's Make the Orchestra Perform!*

---

## Session Overview

**Duration**: 2 hours  
**Format**: 40% Theory + 60% Hands-on Demos  
**Objective**: Transform conceptual understanding into practical Kubernetes skills

**What We Covered in Session 1:**
- ✅ The problem Kubernetes solves
- ✅ Orchestra analogy and core concepts
- ✅ Kubernetes architecture (Cluster → Nodes → Pods → Containers)
- ✅ Pods, Deployments, and Services introduction
- ✅ Basic networking challenge

**Today's Journey:**
- Deep dive into Services with live demos
- Configuration management (ConfigMaps & Secrets)
- Storage and data persistence
- Complete multi-tier application deployment
- Scaling, updates, and rollbacks
- Monitoring and troubleshooting
- Production best practices

---

## Opening: Reconnecting the Orchestra (5 minutes)

### Slide 1: Welcome Back!
**Script**: 
"Welcome back, everyone! In our last session, we learned about the Kubernetes orchestra - the conductor (Kubernetes), the musicians (Pods), the sheet music (Deployments), and the concert hall (Cluster).

Today, we're going backstage. We're going to actually conduct this orchestra ourselves and see the magic happen in real-time. By the end of today's session, you'll be able to deploy, scale, and manage real applications in Kubernetes."

### Slide 2: Quick Recap with Visual
**Visual**: Show the orchestra diagram from Session 1

**Script**: 
"Let's quickly recap the key terms:
- **Cluster**: Our concert hall - the entire infrastructure
- **Nodes**: Orchestra sections - individual servers
- **Pods**: Musicians or small ensembles - containers working together
- **Deployments**: Sheet music - desired state management
- **Services**: The concert hall's address - stable networking

We ended last session talking about the networking challenge. Let's start there and see it in action!"

---

## Section 1: Services Deep Dive (20 minutes)

### Slide 3: The Networking Challenge Revisited
**Script**: 
"Remember this challenge? Each pod gets its own IP address, and these addresses change when pods are recreated. Services solve this problem by providing a stable network endpoint."

### DEMO 1: Creating Pods and Seeing the Networking Problem (Killercoda)
**Killercoda**: Use "Kubernetes Playground"

**Script**: 
"Let's see this networking problem firsthand. I'm going to create a simple web application pod using nginx-alpine which is very lightweight:"

```bash
# Create a pod
kubectl run webapp --image=nginx:alpine --port=80

# Check the pod and note its IP
kubectl get pods -o wide

# Note the IP address (e.g., 10.244.0.5)
```

**Script**: 
"Now watch what happens when I delete and recreate this pod:"

```bash
# Delete the pod
kubectl delete pod webapp

# Recreate it
kubectl run webapp --image=nginx:alpine --port=80

# Check the new IP (wait a few seconds)
kubectl get pods -o wide

# The IP has changed! This is the problem.
```

**Script**: 
"If another application was connecting to the old IP, it would be broken now. This is why we need Services."

### DEMO 2: Creating a Service to Solve the Problem (Killercoda)
**Script**: 
"Let's create a proper deployment with 2 replicas and expose it with a Service:"

```bash
# Create a deployment with 2 replicas (faster than 3)
kubectl create deployment nginx-app --image=nginx:alpine --replicas=2

# Check our pods
kubectl get pods -o wide
# Notice: each pod has a different IP address

# Now expose this deployment with a Service
kubectl expose deployment nginx-app --port=80 --type=ClusterIP

# Check our service
kubectl get service nginx-app

# Describe it to see more details
kubectl describe service nginx-app
```

**Script**: 
"Look at what the Service created:
1. **ClusterIP**: A stable IP that never changes
2. **Endpoints**: The Service automatically tracks all pod IPs
3. **Selector**: Matches pods with label 'app=nginx-app'

Even if pods die and new ones are created with different IPs, the Service ClusterIP stays the same!"

### DEMO 3: Testing Service Discovery (Killercoda)
**Script**: 
"Now let's test service discovery in action. I'll create a test pod and try to connect to our nginx service:"

```bash
# Create a temporary pod to test connectivity using busybox (very fast)
kubectl run test-pod --image=busybox:1.28 --rm -it --restart=Never -- sh

# Inside the test pod (type these commands):
wget -qO- nginx-app
wget -qO- nginx-app
wget -qO- nginx-app

# Type 'exit' to leave
exit
```

**Script**: 
"Did you see that? We used the service name 'nginx-app' directly - no IP addresses! Kubernetes DNS automatically resolves service names. Each request might hit a different pod - that's automatic load balancing!"

### Slide 4: Service Types Explained
**Visual**: Diagram showing different service types

**Script**: 
"Kubernetes offers four types of services, each for different use cases:

**1. ClusterIP (Default)**: 
- Internal only - like an internal phone extension
- Perfect for backend services that shouldn't be exposed outside

**2. NodePort**: 
- Exposes service on each node's IP at a static port
- Like giving your concert hall phone numbers for direct access
- Accessible from outside the cluster

**3. LoadBalancer**: 
- Creates an external load balancer (in cloud environments)
- Like having a professional receptionist directing all calls
- Best for production external services

**4. ExternalName**: 
- Maps service to external DNS name
- Like forwarding calls to an external number"

### DEMO 4: NodePort Service (Killercoda)
**Script**: 
"Let's expose our application externally using NodePort:"

```bash
# Delete the old service first
kubectl delete service nginx-app

# Create a NodePort service
kubectl expose deployment nginx-app --port=80 --type=NodePort

# Get the assigned NodePort
kubectl get service nginx-app

# The output shows something like: 80:32567/TCP
# You can now access it via the Killercoda provided interface
```

**Script**: 
"Now our application is accessible from outside the cluster. In a real cloud environment, you'd access this using the node's public IP and this port."

---

## Section 2: Configuration Management (20 minutes)

### Slide 5: Managing Application Configuration
**Script**: 
"Real applications need configuration - database URLs, API keys, feature flags, environment-specific settings. Hard-coding these in container images is a bad practice. This is where ConfigMaps and Secrets come in.

**ConfigMaps**: Store non-sensitive configuration data
**Secrets**: Store sensitive information like passwords and API keys

In our orchestra analogy, ConfigMaps are like musical arrangements (tempo, key), and Secrets are like the locked combination to the instrument storage."

### DEMO 5: Working with ConfigMaps (Killercoda)
**Script**: 
"Let's create a ConfigMap with application settings:"

```bash
# Create a ConfigMap with literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# View the ConfigMap
kubectl get configmap app-config

# See the contents
kubectl describe configmap app-config

# Get as YAML to see the structure
kubectl get configmap app-config -o yaml
```

**Script**: 
"ConfigMaps are created instantly. Now let's use this configuration in a pod."

### DEMO 6: Using ConfigMaps in Pods (Killercoda)
**Script**: 
"Now let's create a simple pod that uses this ConfigMap:"

```bash
# Create a pod that uses ConfigMap as environment variables
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Configuration loaded:" && env | grep -E "(APP_|LOG_|MAX_)" && sleep 3600']
    envFrom:
    - configMapRef:
        name: app-config
EOF

# Wait a moment for pod to start
sleep 5

# Check the logs to see environment variables
kubectl logs app-with-config
```

**Script**: 
"Perfect! The application now has access to all configuration without any hard-coding. You can update the ConfigMap and restart pods to pick up new values."

### DEMO 7: ConfigMap as Volume Mount (Killercoda)
**Script**: 
"ConfigMaps can also be mounted as files. Let me show you quickly:"

```bash
# First create a config file in a ConfigMap
kubectl create configmap nginx-config --from-literal=index.html="<h1>Hello from ConfigMap!</h1>"

# Now create a pod that mounts it
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-config
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: config
      mountPath: /usr/share/nginx/html
  volumes:
  - name: config
    configMap:
      name: nginx-config
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/nginx-with-config --timeout=30s

# Test it works
kubectl exec nginx-with-config -- cat /usr/share/nginx/html/index.html
```

**Script**: 
"The configuration file is now available inside the container!"

### Slide 6: Secrets - Handling Sensitive Data
**Script**: 
"Now let's talk about Secrets. These work similarly to ConfigMaps but are designed for sensitive information. Kubernetes stores them base64 encoded and can encrypt them at rest."

### DEMO 8: Creating and Using Secrets (Killercoda)
**Script**: 
"Let's create a Secret for database credentials:"

```bash
# Create a Secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123

# View the secret (notice values are hidden)
kubectl get secret db-credentials

# Describe it
kubectl describe secret db-credentials

# Get as YAML (values are base64 encoded)
kubectl get secret db-credentials -o yaml
```

**Script**: 
"Notice the values are base64 encoded. Now let's use it:"

```bash
# Create a pod that uses the secret
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ['sh', '-c', 'echo "DB User: \$DB_USER" && sleep 3600']
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF

# Wait and check the logs
sleep 5
kubectl logs app-with-secrets
```

**Script**: 
"The application can access the credentials, but they're not stored in plain text!"

---

## Section 3: Storage and Persistence (12 minutes)

### Slide 7: The Storage Challenge
**Script**: 
"Until now, all our applications have been stateless. But what about databases? What about user uploads? When a pod dies, all data in it is lost. We need persistent storage.

In our orchestra analogy, this is like the music library - a permanent storage where sheet music is kept safely, even when musicians come and go."

### Slide 8: Volumes, Persistent Volumes, and Claims
**Visual**: Diagram showing Volume types

**Script**: 
"Kubernetes offers several storage solutions:

**1. Volumes**: Temporary storage that lasts the pod's lifetime
**2. Persistent Volumes (PV)**: Cluster-level storage resources
**3. Persistent Volume Claims (PVC)**: Requests for storage by users

Think of it like a library:
- **PV**: The actual library building with shelves
- **PVC**: A library card requesting shelf space
- **Volume**: The actual books mounted in your pod"

### DEMO 9: Using EmptyDir Volume (Killercoda)
**Script**: 
"Let's start with a simple volume that's shared between containers in a pod. EmptyDir provisions instantly:"

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
  - name: writer
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Data from writer" > /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox:1.28
    command: ['sh', '-c', 'sleep 10 && cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

# Wait for containers to run
sleep 15

# Check the reader's logs
kubectl logs shared-storage-pod -c reader
```

**Script**: 
"Both containers can access the same data! But remember, emptyDir is deleted when the pod is deleted. For real persistence, you'd use PersistentVolumes, which I'll demonstrate conceptually."

### DEMO 10: Understanding Persistent Storage Concept
**Script**: 
"Let me show you the YAML structure for persistent storage, though provisioning might take time in free tier:"

```bash
# Show the PVC structure (don't apply if slow)
cat << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-storage
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: data-pvc
EOF
```

**Script**: 
"This is how you request persistent storage. In production cloud environments, this provisions automatically. The key concept: even if the pod is deleted, the PVC and data remain!"

---

## Section 4: Real-World Application Deployment (20 minutes)

### Slide 9: Putting It All Together
**Script**: 
"Now let's deploy a complete multi-tier application using everything we've learned. We'll build a simple but realistic app:
- A Redis database (lightweight, fast to deploy)
- A web frontend
- Services connecting them

This is the grand performance - all orchestra sections working together!"

### DEMO 11: Multi-Tier Application (Killercoda)
**Script**: 
"Let's deploy a complete application stack. We'll use lightweight images for fast provisioning:"

```bash
# Step 1: Deploy Redis (very fast with alpine)
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
EOF

# Wait for Redis to be ready
kubectl wait --for=condition=available deployment/redis --timeout=30s

# Check Redis is running
kubectl get pods -l app=redis
```

**Script**: 
"Step 2: Create configuration for our web app"

```bash
# Create ConfigMap
kubectl create configmap web-config \
  --from-literal=REDIS_HOST=redis \
  --from-literal=APP_NAME="K8s Demo"

# Verify
kubectl get configmap web-config
```

**Script**: 
"Step 3: Deploy a simple web frontend"

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
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
      - name: webapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: web-config
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# Check everything
kubectl get all
```

**Script**: 
"Let's verify everything is working:"

```bash
# Check all pods are running
kubectl get pods

# Check all services
kubectl get svc

# Test internal connectivity
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- webapp
```

**Script**: 
"Our complete application is now running with:
- ✅ Database backend (Redis)
- ✅ Configuration management (ConfigMap)
- ✅ Multiple frontend replicas
- ✅ Service discovery
- ✅ External access via NodePort"

---

## Section 5: Scaling and Updates (18 minutes)

### Slide 10: The Power of Kubernetes - Scaling
**Script**: 
"One of Kubernetes' superpowers is dynamic scaling. Imagine needing more violinists during a crescendo - Kubernetes can add them instantly!"

### DEMO 12: Manual Scaling (Killercoda)
**Script**: 
"Let's scale our application up and down:"

```bash
# Check current replicas
kubectl get deployment webapp

# Scale up to 4 replicas
kubectl scale deployment webapp --replicas=4

# Watch it happen (press Ctrl+C after seeing new pods)
kubectl get pods -l app=webapp -w

# Verify all are running
kubectl get deployment webapp

# Scale back down
kubectl scale deployment webapp --replicas=2

# Watch again
kubectl get pods -l app=webapp
```

**Script**: 
"That's it! We went from 2 to 4 to 2 replicas in seconds. The service automatically updated its endpoints to include all active pods."

### Slide 11: Zero-Downtime Updates
**Script**: 
"Another superpower: updating applications with zero downtime. Kubernetes performs rolling updates - gradually replacing old pods with new ones."

### DEMO 13: Rolling Updates (Killercoda)
**Script**: 
"Let's update our nginx version and watch the rolling update:"

```bash
# Check current image version
kubectl describe deployment webapp | grep Image:

# Update to a newer nginx version
kubectl set image deployment/webapp webapp=nginx:1.25-alpine

# Watch the rollout status
kubectl rollout status deployment/webapp

# See the rollout history
kubectl rollout history deployment/webapp

# Verify new image
kubectl describe deployment webapp | grep Image:
```

**Script**: 
"Did you see that? Kubernetes:
1. Created new pods with the updated image
2. Waited for them to be ready
3. Terminated old pods
4. At no point were we without running instances!"

### DEMO 14: Rollback (Killercoda)
**Script**: 
"What if the new version has a bug? Rollback is just as easy:"

```bash
# Rollback to previous version
kubectl rollout undo deployment/webapp

# Watch the rollback
kubectl rollout status deployment/webapp

# Verify we're back to the previous version
kubectl describe deployment webapp | grep Image:

# Check rollout history
kubectl rollout history deployment/webapp
```

**Script**: 
"In less than a minute, we rolled back to a known good version. This is production-grade deployment capability!"

---

## Section 6: Health Checks and Monitoring (12 minutes)

### Slide 12: Keeping Your Orchestra Healthy
**Script**: 
"A good conductor knows when a musician needs to rest. Kubernetes health checks work the same way:

**Liveness Probe**: Is the application alive? (Restart if not)
**Readiness Probe**: Is the application ready to serve traffic? (Remove from service if not)
**Startup Probe**: Has the application finished starting? (Give slow apps more time)"

### DEMO 15: Adding Health Checks (Killercoda)
**Script**: 
"Let's add health checks to a new deployment:"

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-healthy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-healthy
  template:
    metadata:
      labels:
        app: webapp-healthy
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
EOF

# Watch the deployment
kubectl get deployment webapp-healthy -w

# Check pod health status
kubectl get pods -l app=webapp-healthy
kubectl describe pod -l app=webapp-healthy | grep -A 5 "Liveness\|Readiness"
```

**Script**: 
"Now Kubernetes is continuously checking if our application is healthy and ready to serve traffic!"

### DEMO 16: Basic Monitoring (Killercoda)
**Script**: 
"Let's check resource usage:"

```bash
# Check all pods
kubectl get pods

# Get pod details
kubectl describe pods | grep -A 3 "Conditions:"

# Check events (useful for troubleshooting)
kubectl get events --sort-by='.lastTimestamp' | head -20
```

---

## Section 7: Troubleshooting (10 minutes)

### Slide 13: When Things Go Wrong
**Script**: 
"Even the best orchestra has occasional problems. Let's learn the essential troubleshooting commands."

### DEMO 17: Troubleshooting Toolkit (Killercoda)
**Script**: 
"Here are your essential debugging tools:"

```bash
# 1. Get cluster events
kubectl get events --sort-by='.lastTimestamp'

# 2. Describe resources
kubectl describe pod webapp-healthy

# 3. Check logs
kubectl logs -l app=webapp --tail=20

# 4. Execute commands in running pods
kubectl exec -it deployment/webapp -- sh
# (type 'ls' then 'exit' to leave)

# 5. Check service endpoints
kubectl get endpoints

# 6. Port forward for local testing
kubectl port-forward deployment/webapp 8080:80 &
# Access at http://localhost:8080 (in Killercoda might need different approach)
# Kill the port-forward: pkill -f "port-forward"
```

### DEMO 18: Debugging a Broken Pod (Killercoda)
**Script**: 
"Let's intentionally create a broken pod and troubleshoot it:"

```bash
# Create a pod with a wrong image
kubectl run broken-pod --image=nginx:wrong-version

# Watch it fail
kubectl get pods -w

# (Press Ctrl+C after seeing ImagePullBackOff)

# Check what went wrong
kubectl describe pod broken-pod | grep -A 10 Events:

# Clean up
kubectl delete pod broken-pod
```

**Script**: 
"The error message tells us 'ImagePullBackOff' - the image tag doesn't exist. This is how you diagnose issues in Kubernetes!"

---

## Section 8: Best Practices (8 minutes)

### Slide 14: Production-Ready Kubernetes
**Script**: 
"Before we wrap up, let me share essential best practices for running Kubernetes in production."

### Slide 15: Resource Management
**Script**: 
"Always define resource requests and limits:"

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Slide 16: Key Best Practices
**Script**: 
"Here are the top 10 practices for production:

1. **Always use Deployments**, not bare Pods
2. **Set resource requests and limits** on all containers
3. **Implement health checks** (liveness & readiness probes)
4. **Use ConfigMaps and Secrets** - never hardcode configuration
5. **Use namespaces** to organize environments
6. **Label everything** - makes management easier
7. **Use version control** for all Kubernetes manifests
8. **Implement monitoring** from day one
9. **Plan for disaster recovery** - backup critical data
10. **Use RBAC** - principle of least privilege for security"

### DEMO 19: Production-Ready Template (Killercoda)
**Script**: 
"Here's what a production-ready deployment looks like:"

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  labels:
    app: prod-app
    version: v1.0.0
    environment: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
        version: v1.0.0
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
EOF

kubectl get deployment prod-app
```

**Script**: 
"This deployment includes everything: proper labels, resource limits, health checks!"

---

## Conclusion (8 minutes)

### Slide 17: What We've Accomplished Today
**Script**: 
"Let's recap:

✅ **Services**: Stable networking and load balancing  
✅ **Configuration**: ConfigMaps and Secrets  
✅ **Storage**: Understanding volumes
✅ **Real Application**: Complete multi-tier deployment  
✅ **Scaling**: Dynamic scaling in seconds
✅ **Updates**: Rolling updates and rollbacks  
✅ **Health Checks**: Self-healing capabilities  
✅ **Troubleshooting**: Essential debugging commands  
✅ **Best Practices**: Production-ready configurations  

You've gone from concepts to actually managing applications in Kubernetes!"

### Slide 18: Next Steps & Resources
**Script**: 
"Continue learning:

**This Week**: Practice in Killercoda, deploy your own apps locally

**Next Month**: StatefulSets, Ingress controllers, Helm

**Resources**: kubernetes.io docs, Killercoda scenarios, CNCF community"

### FINAL DEMO: Cleanup (Killercoda)
**Script**: 
"Let's clean up our resources:"

```bash
# Delete deployments
kubectl delete deployment --all

# Delete services
kubectl delete service --all

# Delete pods
kubectl delete pod --all

# Delete configmaps and secrets
kubectl delete configmap --all
kubectl delete secret --all

# Verify cleanup
kubectl get all
```

### Slide 19: Q&A
**Script**: 
"Thank you! The orchestra is yours to conduct now. Any questions?"

---

## Quick Reference for Attendees

### Essential Commands
```bash
# Deployments
kubectl create deployment <name> --image=<image> --replicas=<n>
kubectl scale deployment <name> --replicas=<n>
kubectl set image deployment/<name> <container>=<image>
kubectl rollout undo deployment/<name>

# Services
kubectl expose deployment <name> --port=<p> --type=<type>

# ConfigMaps & Secrets
kubectl create configmap <name> --from-literal=<k>=<v>
kubectl create secret generic <name> --from-literal=<k>=<v>

# Troubleshooting
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- sh
```

---

## Timing
- Opening: 5 min
- Services: 20 min
- Configuration: 20 min
- Storage: 12 min
- Real App: 20 min
- Scaling: 18 min
- Health: 12 min
- Troubleshooting: 10 min
- Best Practices: 8 min
- Conclusion: 8 min
**Total**: ~2 hours 13 min# Kubernetes Session 2: From Concepts to Practice
*Building on Session 1: Now Let's Make the Orchestra Perform!*

---

## Session Overview

**Duration**: 2 hours  
**Format**: 40% Theory + 60% Hands-on Demos  
**Objective**: Transform conceptual understanding into practical Kubernetes skills

**What We Covered in Session 1:**
- ✅ The problem Kubernetes solves
- ✅ Orchestra analogy and core concepts
- ✅ Kubernetes architecture (Cluster → Nodes → Pods → Containers)
- ✅ Pods, Deployments, and Services introduction
- ✅ Basic networking challenge

**Today's Journey:**
- Deep dive into Services with live demos
- Configuration management (ConfigMaps & Secrets)
- Storage and data persistence
- Complete multi-tier application deployment
- Scaling, updates, and rollbacks
- Monitoring and troubleshooting
- Production best practices

---

## Opening: Reconnecting the Orchestra (5 minutes)

### Slide 1: Welcome Back!
**Script**: 
"Welcome back, everyone! In our last session, we learned about the Kubernetes orchestra - the conductor (Kubernetes), the musicians (Pods), the sheet music (Deployments), and the concert hall (Cluster).

Today, we're going backstage. We're going to actually conduct this orchestra ourselves and see the magic happen in real-time. By the end of today's session, you'll be able to deploy, scale, and manage real applications in Kubernetes."

### Slide 2: Quick Recap with Visual
**Visual**: Show the orchestra diagram from Session 1

**Script**: 
"Let's quickly recap the key terms:
- **Cluster**: Our concert hall - the entire infrastructure
- **Nodes**: Orchestra sections - individual servers
- **Pods**: Musicians or small ensembles - containers working together
- **Deployments**: Sheet music - desired state management
- **Services**: The concert hall's address - stable networking

We ended last session talking about the networking challenge. Let's start there and see it in action!"

---

## Section 1: Services Deep Dive (20 minutes)

### Slide 3: The Networking Challenge Revisited
**Script**: 
"Remember this challenge? Each pod gets its own IP address, and these addresses change when pods are recreated. Services solve this problem by providing a stable network endpoint."

### DEMO 1: Creating Pods and Seeing the Networking Problem (Killercoda)
**Killercoda**: Use "Kubernetes Playground"

**Script**: 
"Let's see this networking problem firsthand. I'm going to create a simple web application pod using nginx-alpine which is very lightweight:"

```bash
# Create a pod
kubectl run webapp --image=nginx:alpine --port=80

# Check the pod and note its IP
kubectl get pods -o wide

# Note the IP address (e.g., 10.244.0.5)
```

**Script**: 
"Now watch what happens when I delete and recreate this pod:"

```bash
# Delete the pod
kubectl delete pod webapp

# Recreate it
kubectl run webapp --image=nginx:alpine --port=80

# Check the new IP (wait a few seconds)
kubectl get pods -o wide

# The IP has changed! This is the problem.
```

**Script**: 
"If another application was connecting to the old IP, it would be broken now. This is why we need Services."

### DEMO 2: Creating a Service to Solve the Problem (Killercoda)
**Script**: 
"Let's create a proper deployment with 2 replicas and expose it with a Service:"

```bash
# Create a deployment with 2 replicas (faster than 3)
kubectl create deployment nginx-app --image=nginx:alpine --replicas=2

# Check our pods
kubectl get pods -o wide
# Notice: each pod has a different IP address

# Now expose this deployment with a Service
kubectl expose deployment nginx-app --port=80 --type=ClusterIP

# Check our service
kubectl get service nginx-app

# Describe it to see more details
kubectl describe service nginx-app
```

**Script**: 
"Look at what the Service created:
1. **ClusterIP**: A stable IP that never changes
2. **Endpoints**: The Service automatically tracks all pod IPs
3. **Selector**: Matches pods with label 'app=nginx-app'

Even if pods die and new ones are created with different IPs, the Service ClusterIP stays the same!"

### DEMO 3: Testing Service Discovery (Killercoda)
**Script**: 
"Now let's test service discovery in action. I'll create a test pod and try to connect to our nginx service:"

```bash
# Create a temporary pod to test connectivity using busybox (very fast)
kubectl run test-pod --image=busybox:1.28 --rm -it --restart=Never -- sh

# Inside the test pod (type these commands):
wget -qO- nginx-app
wget -qO- nginx-app
wget -qO- nginx-app

# Type 'exit' to leave
exit
```

**Script**: 
"Did you see that? We used the service name 'nginx-app' directly - no IP addresses! Kubernetes DNS automatically resolves service names. Each request might hit a different pod - that's automatic load balancing!"

### Slide 4: Service Types Explained
**Visual**: Diagram showing different service types

**Script**: 
"Kubernetes offers four types of services, each for different use cases:

**1. ClusterIP (Default)**: 
- Internal only - like an internal phone extension
- Perfect for backend services that shouldn't be exposed outside

**2. NodePort**: 
- Exposes service on each node's IP at a static port
- Like giving your concert hall phone numbers for direct access
- Accessible from outside the cluster

**3. LoadBalancer**: 
- Creates an external load balancer (in cloud environments)
- Like having a professional receptionist directing all calls
- Best for production external services

**4. ExternalName**: 
- Maps service to external DNS name
- Like forwarding calls to an external number"

### DEMO 4: NodePort Service (Killercoda)
**Script**: 
"Let's expose our application externally using NodePort:"

```bash
# Delete the old service first
kubectl delete service nginx-app

# Create a NodePort service
kubectl expose deployment nginx-app --port=80 --type=NodePort

# Get the assigned NodePort
kubectl get service nginx-app

# The output shows something like: 80:32567/TCP
# You can now access it via the Killercoda provided interface
```

**Script**: 
"Now our application is accessible from outside the cluster. In a real cloud environment, you'd access this using the node's public IP and this port."

---

## Section 2: Configuration Management (20 minutes)

### Slide 5: Managing Application Configuration
**Script**: 
"Real applications need configuration - database URLs, API keys, feature flags, environment-specific settings. Hard-coding these in container images is a bad practice. This is where ConfigMaps and Secrets come in.

**ConfigMaps**: Store non-sensitive configuration data
**Secrets**: Store sensitive information like passwords and API keys

In our orchestra analogy, ConfigMaps are like musical arrangements (tempo, key), and Secrets are like the locked combination to the instrument storage."

### DEMO 5: Working with ConfigMaps (Killercoda)
**Script**: 
"Let's create a ConfigMap with application settings:"

```bash
# Create a ConfigMap with literal values
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# View the ConfigMap
kubectl get configmap app-config

# See the contents
kubectl describe configmap app-config

# Get as YAML to see the structure
kubectl get configmap app-config -o yaml
```

**Script**: 
"ConfigMaps are created instantly. Now let's use this configuration in a pod."

### DEMO 6: Using ConfigMaps in Pods (Killercoda)
**Script**: 
"Now let's create a simple pod that uses this ConfigMap:"

```bash
# Create a pod that uses ConfigMap as environment variables
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Configuration loaded:" && env | grep -E "(APP_|LOG_|MAX_)" && sleep 3600']
    envFrom:
    - configMapRef:
        name: app-config
EOF

# Wait a moment for pod to start
sleep 5

# Check the logs to see environment variables
kubectl logs app-with-config
```

**Script**: 
"Perfect! The application now has access to all configuration without any hard-coding. You can update the ConfigMap and restart pods to pick up new values."

### DEMO 7: ConfigMap as Volume Mount (Killercoda)
**Script**: 
"ConfigMaps can also be mounted as files. Let me show you quickly:"

```bash
# First create a config file in a ConfigMap
kubectl create configmap nginx-config --from-literal=index.html="<h1>Hello from ConfigMap!</h1>"

# Now create a pod that mounts it
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-config
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    volumeMounts:
    - name: config
      mountPath: /usr/share/nginx/html
  volumes:
  - name: config
    configMap:
      name: nginx-config
EOF

# Wait for pod to be ready
kubectl wait --for=condition=ready pod/nginx-with-config --timeout=30s

# Test it works
kubectl exec nginx-with-config -- cat /usr/share/nginx/html/index.html
```

**Script**: 
"The configuration file is now available inside the container!"

### Slide 6: Secrets - Handling Sensitive Data
**Script**: 
"Now let's talk about Secrets. These work similarly to ConfigMaps but are designed for sensitive information. Kubernetes stores them base64 encoded and can encrypt them at rest."

### DEMO 8: Creating and Using Secrets (Killercoda)
**Script**: 
"Let's create a Secret for database credentials:"

```bash
# Create a Secret
kubectl create secret generic db-credentials \
  --from-literal=username=admin \
  --from-literal=password=SuperSecret123

# View the secret (notice values are hidden)
kubectl get secret db-credentials

# Describe it
kubectl describe secret db-credentials

# Get as YAML (values are base64 encoded)
kubectl get secret db-credentials -o yaml
```

**Script**: 
"Notice the values are base64 encoded. Now let's use it:"

```bash
# Create a pod that uses the secret
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-secrets
spec:
  containers:
  - name: app
    image: busybox:1.28
    command: ['sh', '-c', 'echo "DB User: \$DB_USER" && sleep 3600']
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: username
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-credentials
          key: password
EOF

# Wait and check the logs
sleep 5
kubectl logs app-with-secrets
```

**Script**: 
"The application can access the credentials, but they're not stored in plain text!"

---

## Section 3: Storage and Persistence (12 minutes)

### Slide 7: The Storage Challenge
**Script**: 
"Until now, all our applications have been stateless. But what about databases? What about user uploads? When a pod dies, all data in it is lost. We need persistent storage.

In our orchestra analogy, this is like the music library - a permanent storage where sheet music is kept safely, even when musicians come and go."

### Slide 8: Volumes, Persistent Volumes, and Claims
**Visual**: Diagram showing Volume types

**Script**: 
"Kubernetes offers several storage solutions:

**1. Volumes**: Temporary storage that lasts the pod's lifetime
**2. Persistent Volumes (PV)**: Cluster-level storage resources
**3. Persistent Volume Claims (PVC)**: Requests for storage by users

Think of it like a library:
- **PV**: The actual library building with shelves
- **PVC**: A library card requesting shelf space
- **Volume**: The actual books mounted in your pod"

### DEMO 9: Using EmptyDir Volume (Killercoda)
**Script**: 
"Let's start with a simple volume that's shared between containers in a pod. EmptyDir provisions instantly:"

```bash
cat << EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: shared-storage-pod
spec:
  containers:
  - name: writer
    image: busybox:1.28
    command: ['sh', '-c', 'echo "Data from writer" > /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  - name: reader
    image: busybox:1.28
    command: ['sh', '-c', 'sleep 10 && cat /data/message.txt && sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data
  volumes:
  - name: shared-data
    emptyDir: {}
EOF

# Wait for containers to run
sleep 15

# Check the reader's logs
kubectl logs shared-storage-pod -c reader
```

**Script**: 
"Both containers can access the same data! But remember, emptyDir is deleted when the pod is deleted. For real persistence, you'd use PersistentVolumes, which I'll demonstrate conceptually."

### DEMO 10: Understanding Persistent Storage Concept
**Script**: 
"Let me show you the YAML structure for persistent storage, though provisioning might take time in free tier:"

```bash
# Show the PVC structure (don't apply if slow)
cat << EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-storage
spec:
  containers:
  - name: app
    image: nginx:alpine
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: data-pvc
EOF
```

**Script**: 
"This is how you request persistent storage. In production cloud environments, this provisions automatically. The key concept: even if the pod is deleted, the PVC and data remain!"

---

## Section 4: Real-World Application Deployment (20 minutes)

### Slide 9: Putting It All Together
**Script**: 
"Now let's deploy a complete multi-tier application using everything we've learned. We'll build a simple but realistic app:
- A Redis database (lightweight, fast to deploy)
- A web frontend
- Services connecting them

This is the grand performance - all orchestra sections working together!"

### DEMO 11: Multi-Tier Application (Killercoda)
**Script**: 
"Let's deploy a complete application stack. We'll use lightweight images for fast provisioning:"

```bash
# Step 1: Deploy Redis (very fast with alpine)
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:alpine
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
EOF

# Wait for Redis to be ready
kubectl wait --for=condition=available deployment/redis --timeout=30s

# Check Redis is running
kubectl get pods -l app=redis
```

**Script**: 
"Step 2: Create configuration for our web app"

```bash
# Create ConfigMap
kubectl create configmap web-config \
  --from-literal=REDIS_HOST=redis \
  --from-literal=APP_NAME="K8s Demo"

# Verify
kubectl get configmap web-config
```

**Script**: 
"Step 3: Deploy a simple web frontend"

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
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
      - name: webapp
        image: nginx:alpine
        ports:
        - containerPort: 80
        envFrom:
        - configMapRef:
            name: web-config
---
apiVersion: v1
kind: Service
metadata:
  name: webapp
spec:
  type: NodePort
  selector:
    app: webapp
  ports:
  - port: 80
    targetPort: 80
EOF

# Check everything
kubectl get all
```

**Script**: 
"Let's verify everything is working:"

```bash
# Check all pods are running
kubectl get pods

# Check all services
kubectl get svc

# Test internal connectivity
kubectl run test --image=busybox:1.28 --rm -it --restart=Never -- wget -qO- webapp
```

**Script**: 
"Our complete application is now running with:
- ✅ Database backend (Redis)
- ✅ Configuration management (ConfigMap)
- ✅ Multiple frontend replicas
- ✅ Service discovery
- ✅ External access via NodePort"

---

## Section 5: Scaling and Updates (18 minutes)

### Slide 10: The Power of Kubernetes - Scaling
**Script**: 
"One of Kubernetes' superpowers is dynamic scaling. Imagine needing more violinists during a crescendo - Kubernetes can add them instantly!"

### DEMO 12: Manual Scaling (Killercoda)
**Script**: 
"Let's scale our application up and down:"

```bash
# Check current replicas
kubectl get deployment webapp

# Scale up to 4 replicas
kubectl scale deployment webapp --replicas=4

# Watch it happen (press Ctrl+C after seeing new pods)
kubectl get pods -l app=webapp -w

# Verify all are running
kubectl get deployment webapp

# Scale back down
kubectl scale deployment webapp --replicas=2

# Watch again
kubectl get pods -l app=webapp
```

**Script**: 
"That's it! We went from 2 to 4 to 2 replicas in seconds. The service automatically updated its endpoints to include all active pods."

### Slide 11: Zero-Downtime Updates
**Script**: 
"Another superpower: updating applications with zero downtime. Kubernetes performs rolling updates - gradually replacing old pods with new ones."

### DEMO 13: Rolling Updates (Killercoda)
**Script**: 
"Let's update our nginx version and watch the rolling update:"

```bash
# Check current image version
kubectl describe deployment webapp | grep Image:

# Update to a newer nginx version
kubectl set image deployment/webapp webapp=nginx:1.25-alpine

# Watch the rollout status
kubectl rollout status deployment/webapp

# See the rollout history
kubectl rollout history deployment/webapp

# Verify new image
kubectl describe deployment webapp | grep Image:
```

**Script**: 
"Did you see that? Kubernetes:
1. Created new pods with the updated image
2. Waited for them to be ready
3. Terminated old pods
4. At no point were we without running instances!"

### DEMO 14: Rollback (Killercoda)
**Script**: 
"What if the new version has a bug? Rollback is just as easy:"

```bash
# Rollback to previous version
kubectl rollout undo deployment/webapp

# Watch the rollback
kubectl rollout status deployment/webapp

# Verify we're back to the previous version
kubectl describe deployment webapp | grep Image:

# Check rollout history
kubectl rollout history deployment/webapp
```

**Script**: 
"In less than a minute, we rolled back to a known good version. This is production-grade deployment capability!"

---

## Section 6: Health Checks and Monitoring (12 minutes)

### Slide 12: Keeping Your Orchestra Healthy
**Script**: 
"A good conductor knows when a musician needs to rest. Kubernetes health checks work the same way:

**Liveness Probe**: Is the application alive? (Restart if not)
**Readiness Probe**: Is the application ready to serve traffic? (Remove from service if not)
**Startup Probe**: Has the application finished starting? (Give slow apps more time)"

### DEMO 15: Adding Health Checks (Killercoda)
**Script**: 
"Let's add health checks to a new deployment:"

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp-healthy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp-healthy
  template:
    metadata:
      labels:
        app: webapp-healthy
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
EOF

# Watch the deployment
kubectl get deployment webapp-healthy -w

# Check pod health status
kubectl get pods -l app=webapp-healthy
kubectl describe pod -l app=webapp-healthy | grep -A 5 "Liveness\|Readiness"
```

**Script**: 
"Now Kubernetes is continuously checking if our application is healthy and ready to serve traffic!"

### DEMO 16: Basic Monitoring (Killercoda)
**Script**: 
"Let's check resource usage:"

```bash
# Check all pods
kubectl get pods

# Get pod details
kubectl describe pods | grep -A 3 "Conditions:"

# Check events (useful for troubleshooting)
kubectl get events --sort-by='.lastTimestamp' | head -20
```

---

## Section 7: Troubleshooting (10 minutes)

### Slide 13: When Things Go Wrong
**Script**: 
"Even the best orchestra has occasional problems. Let's learn the essential troubleshooting commands."

### DEMO 17: Troubleshooting Toolkit (Killercoda)
**Script**: 
"Here are your essential debugging tools:"

```bash
# 1. Get cluster events
kubectl get events --sort-by='.lastTimestamp'

# 2. Describe resources
kubectl describe pod webapp-healthy

# 3. Check logs
kubectl logs -l app=webapp --tail=20

# 4. Execute commands in running pods
kubectl exec -it deployment/webapp -- sh
# (type 'ls' then 'exit' to leave)

# 5. Check service endpoints
kubectl get endpoints

# 6. Port forward for local testing
kubectl port-forward deployment/webapp 8080:80 &
# Access at http://localhost:8080 (in Killercoda might need different approach)
# Kill the port-forward: pkill -f "port-forward"
```

### DEMO 18: Debugging a Broken Pod (Killercoda)
**Script**: 
"Let's intentionally create a broken pod and troubleshoot it:"

```bash
# Create a pod with a wrong image
kubectl run broken-pod --image=nginx:wrong-version

# Watch it fail
kubectl get pods -w

# (Press Ctrl+C after seeing ImagePullBackOff)

# Check what went wrong
kubectl describe pod broken-pod | grep -A 10 Events:

# Clean up
kubectl delete pod broken-pod
```

**Script**: 
"The error message tells us 'ImagePullBackOff' - the image tag doesn't exist. This is how you diagnose issues in Kubernetes!"

---

## Section 8: Best Practices (8 minutes)

### Slide 14: Production-Ready Kubernetes
**Script**: 
"Before we wrap up, let me share essential best practices for running Kubernetes in production."

### Slide 15: Resource Management
**Script**: 
"Always define resource requests and limits:"

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "250m"
  limits:
    memory: "128Mi"
    cpu: "500m"
```

### Slide 16: Key Best Practices
**Script**: 
"Here are the top 10 practices for production:

1. **Always use Deployments**, not bare Pods
2. **Set resource requests and limits** on all containers
3. **Implement health checks** (liveness & readiness probes)
4. **Use ConfigMaps and Secrets** - never hardcode configuration
5. **Use namespaces** to organize environments
6. **Label everything** - makes management easier
7. **Use version control** for all Kubernetes manifests
8. **Implement monitoring** from day one
9. **Plan for disaster recovery** - backup critical data
10. **Use RBAC** - principle of least privilege for security"

### DEMO 19: Production-Ready Template (Killercoda)
**Script**: 
"Here's what a production-ready deployment looks like:"

```bash
cat << EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: prod-app
  labels:
    app: prod-app
    version: v1.0.0
    environment: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: prod-app
  template:
    metadata:
      labels:
        app: prod-app
        version: v1.0.0
    spec:
      containers:
      - name: app
        image: nginx:alpine
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "32Mi"
            cpu: "100m"
          limits:
            memory: "64Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 10
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 3
EOF

kubectl get deployment prod-app
```

**Script**: 
"This deployment includes everything: proper labels, resource limits, health checks!"

---

## Conclusion (8 minutes)

### Slide 17: What We've Accomplished Today
**Script**: 
"Let's recap:

✅ **Services**: Stable networking and load balancing  
✅ **Configuration**: ConfigMaps and Secrets  
✅ **Storage**: Understanding volumes
✅ **Real Application**: Complete multi-tier deployment  
✅ **Scaling**: Dynamic scaling in seconds
✅ **Updates**: Rolling updates and rollbacks  
✅ **Health Checks**: Self-healing capabilities  
✅ **Troubleshooting**: Essential debugging commands  
✅ **Best Practices**: Production-ready configurations  

You've gone from concepts to actually managing applications in Kubernetes!"

### Slide 18: Next Steps & Resources
**Script**: 
"Continue learning:

**This Week**: Practice in Killercoda, deploy your own apps locally

**Next Month**: StatefulSets, Ingress controllers, Helm

**Resources**: kubernetes.io docs, Killercoda scenarios, CNCF community"

### FINAL DEMO: Cleanup (Killercoda)
**Script**: 
"Let's clean up our resources:"

```bash
# Delete deployments
kubectl delete deployment --all

# Delete services
kubectl delete service --all

# Delete pods
kubectl delete pod --all

# Delete configmaps and secrets
kubectl delete configmap --all
kubectl delete secret --all

# Verify cleanup
kubectl get all
```

### Slide 19: Q&A
**Script**: 
"Thank you! The orchestra is yours to conduct now. Any questions?"

---

## Quick Reference for Attendees

### Essential Commands
```bash
# Deployments
kubectl create deployment <name> --image=<image> --replicas=<n>
kubectl scale deployment <name> --replicas=<n>
kubectl set image deployment/<name> <container>=<image>
kubectl rollout undo deployment/<name>

# Services
kubectl expose deployment <name> --port=<p> --type=<type>

# ConfigMaps & Secrets
kubectl create configmap <name> --from-literal=<k>=<v>
kubectl create secret generic <name> --from-literal=<k>=<v>

# Troubleshooting
kubectl get events --sort-by='.lastTimestamp'
kubectl describe pod <name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- sh
```

---

## Timing
- Opening: 5 min
- Services: 20 min
- Configuration: 20 min
- Storage: 12 min
- Real App: 20 min
- Scaling: 18 min
- Health: 12 min
- Troubleshooting: 10 min
- Best Practices: 8 min
- Conclusion: 8 min
**Total**: ~2 hours 13 min