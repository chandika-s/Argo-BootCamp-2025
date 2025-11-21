# Argo Rollout using blue-green deployment

## Prequisite
1. Argocd installation, follow argocd-installation.md
2. Argo Rollout installation, follow argo-rollout.md

## Blue green deployment

1. Install ingress controller
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/aws/deploy.yaml
```
2. Create a file name k8s-manifests\guestbook-rollout.yaml with  the below code

```# 1) Argo Rollout (Blue-Green) with manual promotion
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: guestbook-ui
  namespace: default
spec:
  replicas: 2
  revisionHistoryLimit: 2
  selector:
    matchLabels:
      app: guestbook-ui
  template:
    metadata:
      labels:
        app: guestbook-ui
        version: green     # change to blue/green based on release
    spec:
      containers:
        - name: guestbook-ui
          image: udemykcloud534/guestbook:orange
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
    blueGreen:
      activeService: guestbook-ui-blue
      previewService: guestbook-ui-green
      autoPromotionEnabled: true
      autoPromotionSeconds: 120
      scaleDownDelaySeconds: 300
---
# 2) Blue Service (Active)
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui-blue
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: guestbook-ui
---
# 3) Green Service (Preview)
apiVersion: v1
kind: Service
metadata:
  name: guestbook-ui-green
  namespace: default
spec:
  ports:
    - port: 80
      targetPort: 8080
  selector:
    app: guestbook-ui
---
# 4) Ingress for Blue (/) and Green (/preview)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: guestbook-ui-ingress
  namespace: default
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: guestbook-ui-blue
                port:
                  number: 80
          # regex path captures /preview and everything after it, then rewrites to /<captured>
          - path: /preview(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service:
                name: guestbook-ui-green
                port:
                  number: 80
```

## create application

1. create a file named application.yaml and make sure to verify repoURL matches the repositry which you cloned above.

```
---
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: guestbook-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://github.com/chandika-s/Argo-BootCamp-2025.git'  
    targetRevision: HEAD
    path: k8s-manifests                              
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=false
```

2. apply the application.yaml
```
kubectl apply -f application.yaml
```



## Blue to green deployment

1. Edit the file guestbook-rollout.yaml, change image: udemykcloud534/guestbook:green to image: udemykcloud534/guestbook:blue. commit the changes

2. sync the application from UI

3. Access the blue deployment. 

```
kubectl get ingress -A 
```

3. Perform Blue-Green Deployment. change service guestbook-ui to guestbook-ui-blue-green commit the changes
```
kubectl edit ingress guestbook-ui-ingress -n default 
```


4. promote the deployment

```
kubectl argo rollouts promote guestbook-ui -n default
```


