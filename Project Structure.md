ecommerce/
│── frontend/                     # UI (HTML/JS frontend)
│   ├── index.html
│   ├── Dockerfile
│
│── user-service/                 # User microservice (Node.js)
│   ├── package.json
│   ├── server.js
│   ├── Dockerfile
│
│── product-service/              # Product microservice (Node.js)
│   ├── package.json
│   ├── server.js
│   ├── Dockerfile
│
│── cart-service/                 # Cart microservice (Node.js)
│   ├── package.json
│   ├── server.js
│   ├── Dockerfile
│
│── order-service/                # Order microservice (Node.js)
│   ├── package.json
│   ├── server.js
│   ├── Dockerfile
│
│── k8s/                          # Kubernetes manifests
│   ├── namespace.yaml
│   ├── secrets.yaml
│   ├── configmaps-dbinit.yaml
│   ├── user-postgres-statefulset.yaml
│   ├── product-postgres-statefulset.yaml
│   ├── order-postgres-statefulset.yaml
│   ├── redis-statefulset.yaml
│   │
│   ├── deployments/
│   │   ├── user-deployment.yaml
│   │   ├── product-deployment.yaml
│   │   ├── cart-deployment.yaml
│   │   ├── order-deployment.yaml
│   │   ├── frontend-deployment.yaml
│   │
│   ├── services/
│   │   ├── user-service-svc.yaml
│   │   ├── product-service-svc.yaml
│   │   ├── cart-service-svc.yaml
│   │   ├── order-service-svc.yaml
│   │   ├── frontend-service-svc.yaml
│   │
│   ├── ingress.yaml
