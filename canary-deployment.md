# Canary Deployment

# Prerequisite

1. Install argo cd follow argocd-installation.md
2.  Install Argo Roll out follow argorollout-installation.md


## Install Ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/aws/deploy.yaml
```

## create argo rollout yaml file

1. Create a repositry name argo-rollout-canary and clone it into local
2. Create a file name guestbook-rollout.yaml  within a folder named ** guestbook-rollout **  with the below code
```
# 1) Argo Rollout (Canary) - manual promotion via pause steps
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: guestbook-ui
  namespace: default
spec:
  replicas: 3
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
        version: canary
    spec:
      containers:
        - name: guestbook-ui
          image: udemykcloud534/guestbook:green
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet:
              path: /
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10

  strategy:
    canary:
      trafficRouting:
        nginx:
          stableIngress: guestbook-ui-ingress
          canaryIngress: guestbook-ui-canary-ingress

      steps:
        - setWeight: 10
        - pause:
            duration: 45       # pause for 45 seconds
        - setWeight: 50
        - pause:
            duration: 45       # pause for 45 seconds
        - setWeight: 100
        - pause:
            duration: 45       # pause for 45 seconds (optional)
---
# 2) Stable Service (fronting active/stable ReplicaSet)
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui
  namespace: default
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: guestbook-ui
---
# 3) Canary Service (fronting canary ReplicaSet)
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui-canary
  namespace: default
spec:
  type: ClusterIP
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: guestbook-ui
---
# 4) Stable Ingress -> routes "/" to stable Service
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook-ui-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/rewrite-target: /        # optional: rewrite /preview -> /
    nginx.ingress.kubernetes.io/use-regex: "true"
spec:
  ingressClassName: nginx
  rules:
    - host: guestbook.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: guestbook-ui
                port:
                  number: 80
          - path: /preview(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: guestbook-ui-canary
                port:
                  number: 80
```

## create application

1. create a file named application.yaml and make sure to verify repoURL matches the repositry which you cloned above.

```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-rollout
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/udemykcloud/argo-rollout-canary.git
    path: guestbook-rollout
    targetRevision: HEAD
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated: {}

```
## Modify the guestbook-rollout.yaml for deploying the version for the docker image.

1. Edit the file guestbook-rollout.yaml, change image: udemykcloud534/guestbook:green to image: udemykcloud534/guestbook:blue
2. commit the changes to github
```
git add .
git commit -m "canary deployment"
git push
```


3. apply the application.yaml 

```
kubectl apply -f application.yaml
```
3. Access the loadbalancer dns

```
kubectl get ingress -A
```








