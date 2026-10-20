# Kubernetes Hands-On Projects: Beginner to Advanced

This guide is designed for DevOps training. Start with a local
Kubernetes cluster using **Kind** or **Minikube**. Students can later
repeat the same exercises on Kubernetes in a managed cloud or OpenShift
environment.

> **Lab safety:** Use a disposable cluster and demo data. Do not commit
> credentials to Git. Do not expose dashboards, sample applications, or
> cluster-admin credentials to the public internet.

## Prerequisites

-   Docker installed and running
-   `kubectl`
-   Kind or Minikube
-   Basic familiarity with Docker images, YAML, and Linux commands

Create a Kind cluster:

``` bash
kind create cluster --name k8s-lab
kubectl cluster-info
kubectl get nodes
```

If you use Minikube instead:

``` bash
minikube start
kubectl get nodes
```

------------------------------------------------------------------------

## Project 1: Deploy Your First Application

### Goal

Deploy Nginx, expose it internally with a Service, and access it using
port-forwarding.

Create `deployment.yaml`:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: nginx
          image: nginx:1.28
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
```

Create `service.yaml`:

``` yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
  type: ClusterIP
```

Deploy and test:

``` bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get deployments,pods,services
kubectl rollout status deployment/web
kubectl port-forward service/web 8080:80
```

Open `http://localhost:8080`.

**Learn:** Pods, Deployments, replica count, labels, selectors,
Services, and port-forwarding.

**Challenge:** Scale the Deployment to four replicas and verify the
result.

``` bash
kubectl scale deployment/web --replicas=4
kubectl get pods -o wide
```

------------------------------------------------------------------------

## Project 2: ConfigMaps and Secrets

### Goal

Separate application configuration from the container image and learn
how Kubernetes exposes configuration to a Pod.

Create `config.yaml`:

``` yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
data:
  APP_MESSAGE: "Hello from Kubernetes configuration"
---
apiVersion: v1
kind: Secret
metadata:
  name: demo-secret
type: Opaque
stringData:
  DEMO_PASSWORD: "local-lab-only-change-me"
```

> This Secret is for demonstration only. Kubernetes Secret values are
> not automatically safe just because they are stored as Secrets.
> Configure encryption at rest and RBAC appropriately, and use an
> approved secret manager for real credentials.

Create `app-pod.yaml`:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: shell
      image: busybox:1.36
      command: ["sh", "-c", "echo \"$APP_MESSAGE\"; echo \"Secret is mounted into the environment\"; sleep 3600"]
      env:
        - name: APP_MESSAGE
          valueFrom:
            configMapKeyRef:
              name: web-config
              key: APP_MESSAGE
        - name: DEMO_PASSWORD
          valueFrom:
            secretKeyRef:
              name: demo-secret
              key: DEMO_PASSWORD
```

Run:

``` bash
kubectl apply -f config.yaml
kubectl apply -f app-pod.yaml
kubectl logs config-demo
kubectl describe pod config-demo
```

Avoid printing the actual secret value into logs or terminal history.

**Challenge:** Update the ConfigMap and compare how environment-variable
configuration behaves in an already-running Pod versus a newly created
Pod.

------------------------------------------------------------------------

## Project 3: Health Checks and Self-Healing

### Goal

Configure readiness and liveness probes and observe how Kubernetes
reacts when an application is not healthy.

Create `health-deployment.yaml`:

``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: health-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: health-demo
  template:
    metadata:
      labels:
        app: health-demo
    spec:
      containers:
        - name: nginx
          image: nginx:1.28
          ports:
            - containerPort: 80
          readinessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /
              port: 80
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 250m
              memory: 128Mi
```

Run:

``` bash
kubectl apply -f health-deployment.yaml
kubectl rollout status deployment/health-demo
kubectl get pods
kubectl describe pods -l app=health-demo
```

**Learn:** - Readiness determines whether a Pod should receive Service
traffic. - Liveness determines whether Kubernetes should restart a
container. - Startup probes can be useful for slow-starting
applications.

**Challenge:** Change the probe path to a nonexistent path and observe
events and readiness. Restore the correct path afterward.

------------------------------------------------------------------------

## Project 4: Rolling Updates and Rollbacks

### Goal

Release a new application version with minimal disruption and practise
rollback.

``` bash
kubectl set image deployment/web nginx=nginx:1.28
kubectl rollout status deployment/web
kubectl rollout history deployment/web
```

For a clear version-change exercise, use two image tags available in
your environment, and record the original image first:

``` bash
kubectl get deployment web -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'
```

Then set the new image tag and observe the rollout:

``` bash
kubectl set image deployment/web nginx=nginx:1.29
kubectl rollout status deployment/web
kubectl rollout history deployment/web
```

If the chosen image tag is unavailable, use a valid tag supported by
your environment. Do not use a made-up tag.

Rollback:

``` bash
kubectl rollout undo deployment/web
kubectl rollout status deployment/web
```

**Challenge:** Set an invalid image tag, observe the rollout problem,
then roll back.

------------------------------------------------------------------------

## Project 5: Persistent Storage with a PVC

### Goal

Understand why data that must survive Pod replacement should be stored
outside the container filesystem.

Create `storage.yaml`:

``` yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: lab-data
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
  name: storage-demo
spec:
  containers:
    - name: writer
      image: busybox:1.36
      command: ["sh", "-c", "while true; do date >> /data/timestamps.txt; sleep 10; done"]
      volumeMounts:
        - name: data
          mountPath: /data
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: lab-data
```

Apply and inspect:

``` bash
kubectl apply -f storage.yaml
kubectl get pvc,pod
kubectl exec storage-demo -- tail /data/timestamps.txt
```

Delete and recreate the Pod (not the PVC):

``` bash
kubectl delete pod storage-demo
kubectl apply -f storage.yaml
kubectl exec storage-demo -- tail /data/timestamps.txt
```

The PVC requires a compatible StorageClass or provisioner. Check with
`kubectl get storageclass`; if the claim remains `Pending`, investigate
the cluster's storage configuration.

**Challenge:** Explain the difference between container filesystem,
`emptyDir`, PVC, and a cloud-backed PersistentVolume.

------------------------------------------------------------------------

## Project 6: Ingress or Gateway-Based Routing

### Goal

Route HTTP traffic to a Service using hostnames.

Ingress resources require an Ingress controller; creating the YAML alone
does not install one. On Kind, install a controller using its current
official installation instructions, or use Minikube's supported ingress
addon.

Example `ingress.yaml` (assuming an Ingress controller is installed):

``` yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: web.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: web
                port:
                  number: 80
```

Apply and inspect:

``` bash
kubectl apply -f ingress.yaml
kubectl get ingress
kubectl describe ingress web-ingress
```

Configure `web.local` to resolve to the controller's reachable IP
according to your local cluster setup. If you do not want to configure
hostnames, continue using `kubectl port-forward`.

**Challenge:** Add a second application and route it using a different
hostname.

------------------------------------------------------------------------

## Project 7: Autoscaling with Horizontal Pod Autoscaler

### Goal

Scale replicas based on CPU utilization.

A metrics API (commonly Metrics Server) must be available, and the
Deployment needs CPU requests for utilization-based scaling.

For the `web` Deployment from Project 1, run:

``` bash
kubectl autoscale deployment web --cpu-percent=60 --min=2 --max=6
kubectl get hpa
kubectl describe hpa web
```

If Metrics Server is not installed or working, `kubectl get hpa` may
show unknown CPU metrics. Install and configure it using the
instructions for your cluster, then verify:

``` bash
kubectl top nodes
kubectl top pods
```

**Challenge:** Generate test traffic, observe CPU metrics, and describe
why an HPA may not scale immediately. Clean up the autoscaler afterward:

``` bash
kubectl delete hpa web
```

------------------------------------------------------------------------

## Project 8: Resource Requests, Limits, and Troubleshooting

### Goal

Understand scheduling and common workload failures.

Inspect a workload:

``` bash
kubectl get pods
kubectl describe pod <POD_NAME>
kubectl logs <POD_NAME>
kubectl get events --sort-by=.metadata.creationTimestamp
kubectl top pods
kubectl top nodes
```

Common states to investigate:

-   `Pending`: scheduling constraints, unavailable resources, or storage
    issues.
-   `ImagePullBackOff`: wrong image name/tag, registry access, or
    authentication.
-   `CrashLoopBackOff`: the process repeatedly exits; inspect logs and
    configuration.
-   `OOMKilled`: the container exceeded its memory limit.
-   `0/1 Ready`: readiness probe failure or app not listening correctly.

**Challenge:** Induce one safe failure at a time, diagnose it from
events/logs, and write a short incident report with root cause and
remediation.

------------------------------------------------------------------------

## Project 9: Deploy an Application Using Helm

### Goal

Package reusable Kubernetes manifests in a Helm chart and override
values between environments.

Install Helm using the official instructions for your operating system.

Create a chart scaffold:

``` bash
helm create web-chart
```

Inspect the chart files and values:

``` bash
helm lint web-chart
helm template web-lab ./web-chart
```

Install into a dedicated namespace:

``` bash
kubectl create namespace helm-lab
helm install web-lab ./web-chart -n helm-lab
helm list -n helm-lab
kubectl get all -n helm-lab
```

Upgrade with a values override or chart change:

``` bash
helm upgrade web-lab ./web-chart -n helm-lab
helm history web-lab -n helm-lab
helm rollback web-lab 1 -n helm-lab
```

The generated chart contains example templates. Review them and remove
unnecessary resources before using the chart for a real app.

**Challenge:** Use separate values files for `dev` and `staging`, then
show the rendered differences with `helm template`.

------------------------------------------------------------------------

## Project 10: CI/CD from Git to Kubernetes

### Goal

Build a simple delivery flow: commit code, run tests, build an image,
publish it to a registry, and deploy to Kubernetes.

Suggested pipeline stages:

1.  Validate manifests or chart.
2.  Run unit tests.
3.  Build a container image with a supported builder.
4.  Push an immutable tag to a registry.
5.  Deploy to a dedicated test namespace.
6.  Wait for rollout completion.
7.  Run a smoke test and preserve logs on failure.

Example validation commands:

``` bash
kubectl apply --dry-run=client -f k8s/
helm lint ./web-chart
helm template web-lab ./web-chart
```

These validate manifests or templates; they do not prove the application
will run successfully in the cluster.

Before adding a deployment job: - Use a dedicated CI identity with
least-privilege RBAC. - Store credentials in the CI platform's protected
secret store. - Do not commit kubeconfig files or administrator
tokens. - Ensure the runner can reach the target cluster and registry. -
Restrict production deployments to approved branches/tags and use
environment approvals where appropriate.

**Challenge:** Deploy a test version, deliberately fail the smoke test,
and demonstrate how the pipeline reports the failure and how the team
rolls back safely.

------------------------------------------------------------------------

## Optional Advanced Project: Deploy on OpenShift

If your training environment is OpenShift, repeat Projects 1, 2, 3, 4,
5, 8, 9, and 10 on a non-production project.

Explore: - Projects and RBAC - Routes instead of assuming a generic
Ingress controller - OpenShift SCC and non-root container requirements -
ImageStreams and integrated image registry where available -
ResourceQuota and LimitRange - OpenShift monitoring and logs - GitOps
with OpenShift GitOps/Argo CD, if installed

Do not assume every generic Kubernetes manifest or container image will
work unchanged on OpenShift. Validate security context, storage,
routing, and cluster policy for the target environment.

## Cleanup

Delete individual lab resources when finished, or remove the disposable
Kind cluster:

``` bash
kind delete cluster --name k8s-lab
```

For Minikube:

``` bash
minikube delete
```

## Suggested Teaching Order

  Session   Project                  Main learning
  --------- ------------------------ -----------------------------------
  1         First application        Pods, Deployments, Services
  2         ConfigMaps and Secrets   Configuration and secret handling
  3         Health checks            Readiness, liveness, self-healing
  4         Rollouts                 Updates and rollback
  5         Persistent storage       PVCs and data lifecycle
  6         Ingress                  HTTP routing
  7         Autoscaling              Metrics and HPA
  8         Troubleshooting          Events, logs, resources
  9         Helm                     Reusable application packaging
  10        CI/CD                    Automated delivery to Kubernetes

## Final Outcome

By the end of these labs, learners should be able to deploy an
application, configure it safely, expose it through a Service or ingress
layer, persist data, scale and troubleshoot workloads, package manifests
with Helm, and understand the building blocks of a CI/CD deployment to
Kubernetes or OpenShift.
