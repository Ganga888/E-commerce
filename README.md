# E-Commerce Microservices on Kubernetes (kOps) — `ganga888.online`

This repo contains a minimal, production-shaped e-commerce app modeled after Flipkart/Amazon with **four microservices** and a tiny frontend. It’s built to run on a **kOps** Kubernetes cluster (1 control-plane, 3 workers) and exposes the app at **https://ganga888.online**.

> ✅ You asked for **one database per microservice** and **JWT auth** — implemented below.  
> ✅ All workloads run with **3 replicas** and **pod anti-affinity** to spread across the 3 worker nodes.

---

## 0) Architecture at a glance

**Services**
- `user-service` — signup/login, issues JWT. DB: **Postgres (userdb)**
- `product-service` — product catalog (list/details). DB: **Postgres (productdb)**
- `cart-service` — per-user cart stored in **Redis**
- `order-service` — checkout + order history. DB: **Postgres (orderdb)**
- `frontend` — static HTML/JS calling the APIs

**Kubernetes components & why we use them**
- **Namespace** (`ecommerce`) — logical isolation
- **Secret** — JWT secret + DB passwords (sensitive)
- **ConfigMap** — DB init SQL files (non-sensitive)
- **StatefulSet + PVC** — Postgres & Redis persistent storage and stable identity
- **Deployment** — stateless app pods (3 replicas each)
- **Service (ClusterIP)** — stable in-cluster virtual IP per service
- **Ingress** — single external entry (`ganga888.online`) routing to services
- **Probes** — liveness/readiness to keep only healthy pods in rotation
- **podAntiAffinity** — spread replicas across worker nodes for resilience

---

## 1) Prerequisites

- A working **kOps** cluster on AWS (or similar) with `kubectl` access.
- **Helm** installed locally.
- A container registry (Docker Hub or AWS ECR).
- Control of the DNS zone for **ganga888.online** (Route 53, Cloudflare, etc.).

> If your cluster already exists, you can skip any kOps setup and continue.

---

## 2) Repo layout (create these folders/files)

ecommerce/
frontend/
index.html
Dockerfile
user-service/
package.json
server.js
Dockerfile
product-service/
package.json
server.js
Dockerfile
cart-service/
package.json
server.js
Dockerfile
order-service/
package.json
server.js
Dockerfile
k8s/
namespace.yaml
secrets.yaml
configmaps-dbinit.yaml
user-postgres-statefulset.yaml
product-postgres-statefulset.yaml
order-postgres-statefulset.yaml
redis-statefulset.yaml
deployments/
user-deployment.yaml
product-deployment.yaml
cart-deployment.yaml
order-deployment.yaml
frontend-deployment.yaml
services/
user-service-svc.yaml
product-service-svc.yaml
cart-service-svc.yaml
order-service-svc.yaml
frontend-service-svc.yaml
ingress.yaml

pgsql
Copy code

---

## 3) Application code & Dockerfiles

> Replace **`<your-registry>`** everywhere with your registry (e.g., `docker.io/yourname` or your ECR URI).

### 3.1 `user-service` (JWT auth, Postgres)
**`user-service/package.json`**
```json
{
  "name": "user-service",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "bcryptjs": "^2.4.3",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "pg": "^8.11.0"
  }
}
user-service/server.js

js
Copy code
const express = require('express');
const { Pool } = require('pg');
const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

const pool = new Pool({
  host: process.env.DB_HOST || 'user-postgres.ecommerce.svc.cluster.local',
  port: process.env.DB_PORT || 5432,
  user: process.env.DB_USER || 'user',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'userdb'
});

const JWT_SECRET = process.env.JWT_SECRET || 'devsecret';
const app = express();
app.use(express.json());

app.get('/healthz', (req, res) => res.send('ok'));

app.post('/register', async (req, res) => {
  const { username, password } = req.body;
  if (!username || !password) return res.status(400).json({error: 'username+password required'});
  const hash = bcrypt.hashSync(password, 8);
  try {
    const r = await pool.query(
      'INSERT INTO users (username, password_hash) VALUES ($1,$2) RETURNING id, username',
      [username, hash]
    );
    res.json({ user: r.rows[0] });
  } catch (e) {
    if (e.code === '23505') return res.status(409).json({ error: 'username exists' });
    console.error(e); res.status(500).json({ error: 'db error' });
  }
});

app.post('/login', async (req, res) => {
  const { username, password } = req.body;
  const r = await pool.query('SELECT id, username, password_hash FROM users WHERE username=$1', [username]);
  const user = r.rows[0];
  if (!user || !bcrypt.compareSync(password, user.password_hash)) return res.status(401).json({ error: 'invalid credentials' });
  const token = jwt.sign({ userId: user.id, username: user.username }, JWT_SECRET, { expiresIn: '2h' });
  res.json({ token });
});

function authenticate(req, res, next) {
  const h = req.headers.authorization;
  if (!h) return res.status(401).json({ error: 'missing auth' });
  try { req.user = jwt.verify(h.split(' ')[1], JWT_SECRET); next(); }
  catch { return res.status(401).json({ error: 'invalid token' }); }
}

app.get('/me', authenticate, async (req, res) => {
  const r = await pool.query('SELECT id, username, created_at FROM users WHERE id=$1', [req.user.userId]);
  res.json({ user: r.rows[0] });
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log('user-service listening', port));
user-service/Dockerfile

dockerfile
Copy code
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node","server.js"]
3.2 product-service (Postgres)
product-service/package.json

json
Copy code
{
  "name": "product-service",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.18.2",
    "pg": "^8.11.0"
  }
}
product-service/server.js

js
Copy code
const express = require('express');
const { Pool } = require('pg');
const app = express();
app.use(express.json());

const pool = new Pool({
  host: process.env.DB_HOST || 'product-postgres.ecommerce.svc.cluster.local',
  user: process.env.DB_USER || 'product',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'productdb',
  port: process.env.DB_PORT || 5432
});

app.get('/healthz', (req, res) => res.send('ok'));
app.get('/products', async (req, res) => {
  const r = await pool.query('SELECT id, name, description, price FROM products ORDER BY id');
  res.json(r.rows);
});
app.get('/products/:id', async (req, res) => {
  const r = await pool.query('SELECT id, name, description, price FROM products WHERE id=$1', [req.params.id]);
  if (r.rowCount===0) return res.status(404).json({ error: 'not found' });
  res.json(r.rows[0]);
});
app.post('/products', async (req, res) => {
  const { name, description, price } = req.body;
  const r = await pool.query('INSERT INTO products (name, description, price) VALUES ($1,$2,$3) RETURNING id', [name, description, price]);
  res.json({ id: r.rows[0].id });
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log('product-service listening', port));
product-service/Dockerfile

dockerfile
Copy code
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node","server.js"]
3.3 cart-service (Redis, JWT)
cart-service/package.json

json
Copy code
{
  "name": "cart-service",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "express": "^4.18.2",
    "ioredis": "^5.3.2",
    "jsonwebtoken": "^9.0.0"
  }
}
cart-service/server.js

js
Copy code
const express = require('express');
const Redis = require('ioredis');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());

const redis = new Redis({
  host: process.env.REDIS_HOST || 'cart-redis.ecommerce.svc.cluster.local',
  port: process.env.REDIS_PORT || 6379
});

const JWT_SECRET = process.env.JWT_SECRET || 'devsecret';

function authenticate(req, res, next) {
  const h = req.headers.authorization;
  if (!h) return res.status(401).json({ error: 'missing auth' });
  try { req.user = jwt.verify(h.split(' ')[1], JWT_SECRET); next(); }
  catch { return res.status(401).json({ error: 'invalid token' }); }
}

const keyFor = userId => `cart:${userId}`;

app.post('/cart/add', authenticate, async (req, res) => {
  const { productId, quantity } = req.body;
  if (!productId || !quantity) return res.status(400).json({ error: 'productId+quantity required' });
  const key = keyFor(req.user.userId);
  const raw = await redis.get(key);
  const cart = raw ? JSON.parse(raw) : [];
  const i = cart.findIndex(it => it.productId == productId);
  if (i >= 0) cart[i].quantity += quantity; else cart.push({ productId, quantity });
  await redis.set(key, JSON.stringify(cart));
  res.json({ status: 'ok', cart });
});

app.get('/cart', authenticate, async (req, res) => {
  const raw = await redis.get(keyFor(req.user.userId));
  res.json({ cart: raw ? JSON.parse(raw) : [] });
});

app.post('/cart/remove', authenticate, async (req, res) => {
  const key = keyFor(req.user.userId);
  const raw = await redis.get(key);
  let cart = raw ? JSON.parse(raw) : [];
  cart = cart.filter(c => c.productId != req.body.productId);
  await redis.set(key, JSON.stringify(cart));
  res.json({ cart });
});

app.post('/cart/clear', authenticate, async (req, res) => {
  await redis.del(keyFor(req.user.userId));
  res.json({ status: 'cleared' });
});

app.get('/healthz', (req, res) => res.send('ok'));

const port = process.env.PORT || 3000;
app.listen(port, () => console.log('cart-service listening', port));
cart-service/Dockerfile

dockerfile
Copy code
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node","server.js"]
3.4 order-service (Postgres, calls Cart + Product, JWT)
order-service/package.json

json
Copy code
{
  "name": "order-service",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": { "start": "node server.js" },
  "dependencies": {
    "axios": "^1.4.0",
    "express": "^4.18.2",
    "jsonwebtoken": "^9.0.0",
    "pg": "^8.11.0"
  }
}
order-service/server.js

js
Copy code
const express = require('express');
const { Pool } = require('pg');
const axios = require('axios');
const jwt = require('jsonwebtoken');

const app = express();
app.use(express.json());

const pool = new Pool({
  host: process.env.DB_HOST || 'order-postgres.ecommerce.svc.cluster.local',
  user: process.env.DB_USER || 'order',
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME || 'orderdb',
  port: process.env.DB_PORT || 5432
});

const JWT_SECRET = process.env.JWT_SECRET || 'devsecret';
function authenticate(req,res,next){
  const h = req.headers.authorization; if(!h) return res.status(401).json({error:'missing auth'});
  try { req.user = jwt.verify(h.split(' ')[1], JWT_SECRET); next(); } catch { return res.status(401).json({error:'invalid token'}) }
}

app.get('/healthz', (req,res)=>res.send('ok'));

app.post('/orders', authenticate, async (req, res) => {
  const authHeader = req.headers.authorization;
  const cartRes = await axios.get('http://cart-service.ecommerce.svc.cluster.local/cart', { headers: { Authorization: authHeader }});
  const cart = cartRes.data.cart || [];
  if (cart.length === 0) return res.status(400).json({ error: 'cart empty' });

  const items = [];
  for (const it of cart) {
    const p = await axios.get(`http://product-service.ecommerce.svc.cluster.local/products/${it.productId}`);
    items.push({ productId: it.productId, quantity: it.quantity, price: parseFloat(p.data.price) });
  }
  const total = items.reduce((s,i)=> s + i.price * i.quantity, 0);

  try {
    await pool.query('BEGIN');
    const r = await pool.query('INSERT INTO orders (user_id, total) VALUES ($1,$2) RETURNING id', [req.user.userId, total]);
    const orderId = r.rows[0].id;
    for (const it of items) {
      await pool.query('INSERT INTO order_items (order_id, product_id, quantity, price_at_purchase) VALUES ($1,$2,$3,$4)',
        [orderId, it.productId, it.quantity, it.price]);
    }
    await pool.query('COMMIT');
    await axios.post('http://cart-service.ecommerce.svc.cluster.local/cart/clear', {}, { headers: { Authorization: authHeader }});
    res.json({ orderId, total });
  } catch (e) {
    await pool.query('ROLLBACK'); console.error(e); res.status(500).json({ error: 'order failed' });
  }
});

app.get('/orders', authenticate, async (req, res) => {
  const r = await pool.query('SELECT id, total, created_at FROM orders WHERE user_id=$1 ORDER BY created_at DESC', [req.user.userId]);
  res.json({ orders: r.rows });
});

const port = process.env.PORT || 3000;
app.listen(port, () => console.log('order-service listening', port));
order-service/Dockerfile

dockerfile
Copy code
FROM node:18-alpine
WORKDIR /app
COPY package.json .
RUN npm install --production
COPY . .
EXPOSE 3000
CMD ["node","server.js"]
3.5 frontend (static HTML)
frontend/index.html

html
Copy code
<!doctype html>
<html>
<head><meta charset="utf-8"><title>Mini Shop</title></head>
<body>
<h1>Mini Shop (Demo)</h1>
<div id="auth">
  <input id="u" placeholder="username"/><input id="p" placeholder="password" type="password"/>
  <button onclick="register()">Register</button>
  <button onclick="login()">Login</button>
</div>
<div id="actions" style="display:none">
  <button onclick="listProducts()">List Products</button>
  <button onclick="viewCart()">View Cart</button>
  <button onclick="addToCart()">Add 1st Product (x2)</button>
  <button onclick="checkout()">Checkout</button>
  <pre id="out"></pre>
</div>
<script>
let token = '';
async function register() {
  const u=document.getElementById('u').value; const p=document.getElementById('p').value;
  const r=await fetch('/api/user/register',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:u,password:p})});
  document.getElementById('out').innerText = await r.text();
}
async function login() {
  const u=document.getElementById('u').value; const p=document.getElementById('p').value;
  const r=await fetch('/api/user/login',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({username:u,password:p})});
  const j=await r.json(); token=j.token; if(token){document.getElementById('actions').style.display='block';}
  document.getElementById('out').innerText = JSON.stringify(j,null,2);
}
async function listProducts() {
  const r=await fetch('/api/product/products'); const j=await r.json();
  document.getElementById('out').innerText = JSON.stringify(j,null,2);
}
async function addToCart() {
  const r=await fetch('/api/product/products'); const products=await r.json();
  if(!products.length){document.getElementById('out').innerText='No products';return;}
  const first=products[0];
  const res=await fetch('/api/cart/cart/add',{method:'POST',headers:{'Content-Type':'application/json','Authorization':'Bearer '+token},body:JSON.stringify({productId:first.id,quantity:2})});
  document.getElementById('out').innerText = await res.text();
}
async function viewCart() {
  const r=await fetch('/api/cart/cart',{headers:{Authorization:'Bearer '+token}});
  const j=await r.json(); document.getElementById('out').innerText = JSON.stringify(j,null,2);
}
async function checkout() {
  const r=await fetch('/api/order/orders',{method:'POST',headers:{Authorization:'Bearer '+token}});
  const j=await r.json(); document.getElementById('out').innerText = JSON.stringify(j,null,2);
}
</script>
</body>
</html>
frontend/Dockerfile

dockerfile
Copy code
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
CMD ["nginx","-g","daemon off;"]
4) Kubernetes manifests
4.1 Common
k8s/namespace.yaml

yaml
Copy code
apiVersion: v1
kind: Namespace
metadata: { name: ecommerce }
k8s/secrets.yaml (edit passwords & secret!)

yaml
Copy code
apiVersion: v1
kind: Secret
metadata: { name: ecommerce-secrets, namespace: ecommerce }
type: Opaque
stringData:
  JWT_SECRET: "REPLACE_WITH_STRONG_JWT_SECRET"
  USER_DB_PASSWORD: "user_db_pass"
  PRODUCT_DB_PASSWORD: "product_db_pass"
  ORDER_DB_PASSWORD: "order_db_pass"
k8s/configmaps-dbinit.yaml

yaml
Copy code
apiVersion: v1
kind: ConfigMap
metadata: { name: db-init-sql, namespace: ecommerce }
data:
  user-init.sql: |
    CREATE TABLE IF NOT EXISTS users (
      id SERIAL PRIMARY KEY,
      username TEXT UNIQUE NOT NULL,
      password_hash TEXT NOT NULL,
      created_at TIMESTAMP DEFAULT now()
    );
  product-init.sql: |
    CREATE TABLE IF NOT EXISTS products (
      id SERIAL PRIMARY KEY,
      name TEXT NOT NULL,
      description TEXT,
      price NUMERIC(10,2) NOT NULL,
      created_at TIMESTAMP DEFAULT now()
    );
    INSERT INTO products (name, description, price) VALUES
      ('Sample Product A','Desc A', 19.99),
      ('Sample Product B','Desc B', 29.50)
    ON CONFLICT DO NOTHING;
  order-init.sql: |
    CREATE TABLE IF NOT EXISTS orders (
      id SERIAL PRIMARY KEY,
      user_id INTEGER NOT NULL,
      total NUMERIC(10,2) NOT NULL,
      created_at TIMESTAMP DEFAULT now()
    );
    CREATE TABLE IF NOT EXISTS order_items (
      id SERIAL PRIMARY KEY,
      order_id INTEGER REFERENCES orders(id) ON DELETE CASCADE,
      product_id INTEGER NOT NULL,
      quantity INTEGER NOT NULL,
      price_at_purchase NUMERIC(10,2) NOT NULL
    );
4.2 Databases (StatefulSets)
Adjust storageClassName to your cluster (e.g., gp2/gp3 for AWS EBS).

k8s/user-postgres-statefulset.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: user-postgres, namespace: ecommerce, labels: { app: user-postgres } }
spec:
  ports: [{ name: postgres, port: 5432 }]
  clusterIP: None
  selector: { app: user-postgres }
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: user-postgres, namespace: ecommerce }
spec:
  serviceName: "user-postgres"
  replicas: 1
  selector: { matchLabels: { app: user-postgres } }
  template:
    metadata: { labels: { app: user-postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - { name: POSTGRES_DB,  value: userdb }
            - { name: POSTGRES_USER, value: user }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: USER_DB_PASSWORD } }
          ports: [{ containerPort: 5432 }]
          volumeMounts:
            - { name: pgdata,  mountPath: /var/lib/postgresql/data }
            - { name: init-sql, mountPath: /docker-entrypoint-initdb.d/user-init.sql, subPath: user-init.sql }
  volumeClaimTemplates:
    - metadata: { name: pgdata }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 5Gi } }
        storageClassName: gp2
  volumeMounts: []
  volumeClaimTemplates: []
  template:
    spec:
      volumes:
        - name: init-sql
          configMap:
            name: db-init-sql
            items: [{ key: user-init.sql, path: user-init.sql }]
k8s/product-postgres-statefulset.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: product-postgres, namespace: ecommerce, labels: { app: product-postgres } }
spec:
  ports: [{ name: postgres, port: 5432 }]
  clusterIP: None
  selector: { app: product-postgres }
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: product-postgres, namespace: ecommerce }
spec:
  serviceName: "product-postgres"
  replicas: 1
  selector: { matchLabels: { app: product-postgres } }
  template:
    metadata: { labels: { app: product-postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - { name: POSTGRES_DB,  value: productdb }
            - { name: POSTGRES_USER, value: product }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: PRODUCT_DB_PASSWORD } }
          ports: [{ containerPort: 5432 }]
          volumeMounts:
            - { name: pgdata,  mountPath: /var/lib/postgresql/data }
            - { name: init-sql, mountPath: /docker-entrypoint-initdb.d/product-init.sql, subPath: product-init.sql }
  volumeClaimTemplates:
    - metadata: { name: pgdata }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 5Gi } }
        storageClassName: gp2
  template:
    spec:
      volumes:
        - name: init-sql
          configMap:
            name: db-init-sql
            items: [{ key: product-init.sql, path: product-init.sql }]
k8s/order-postgres-statefulset.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: order-postgres, namespace: ecommerce, labels: { app: order-postgres } }
spec:
  ports: [{ name: postgres, port: 5432 }]
  clusterIP: None
  selector: { app: order-postgres }
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: order-postgres, namespace: ecommerce }
spec:
  serviceName: "order-postgres"
  replicas: 1
  selector: { matchLabels: { app: order-postgres } }
  template:
    metadata: { labels: { app: order-postgres } }
    spec:
      containers:
        - name: postgres
          image: postgres:15-alpine
          env:
            - { name: POSTGRES_DB,  value: orderdb }
            - { name: POSTGRES_USER, value: order }
            - name: POSTGRES_PASSWORD
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: ORDER_DB_PASSWORD } }
          ports: [{ containerPort: 5432 }]
          volumeMounts:
            - { name: pgdata,  mountPath: /var/lib/postgresql/data }
            - { name: init-sql, mountPath: /docker-entrypoint-initdb.d/order-init.sql, subPath: order-init.sql }
  volumeClaimTemplates:
    - metadata: { name: pgdata }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 5Gi } }
        storageClassName: gp2
  template:
    spec:
      volumes:
        - name: init-sql
          configMap:
            name: db-init-sql
            items: [{ key: order-init.sql, path: order-init.sql }]
k8s/redis-statefulset.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: cart-redis, namespace: ecommerce, labels: { app: cart-redis } }
spec:
  ports: [{ name: redis, port: 6379 }]
  clusterIP: None
  selector: { app: cart-redis }
---
apiVersion: apps/v1
kind: StatefulSet
metadata: { name: cart-redis, namespace: ecommerce }
spec:
  serviceName: "cart-redis"
  replicas: 1
  selector: { matchLabels: { app: cart-redis } }
  template:
    metadata: { labels: { app: cart-redis } }
    spec:
      containers:
        - name: redis
          image: redis:7-alpine
          ports: [{ containerPort: 6379 }]
          volumeMounts:
            - { name: redisdata, mountPath: /data }
  volumeClaimTemplates:
    - metadata: { name: redisdata }
      spec:
        accessModes: ["ReadWriteOnce"]
        resources: { requests: { storage: 2Gi } }
        storageClassName: gp2
4.3 Deployments (3 replicas + anti-affinity)
Copy the anti-affinity block across all deployments.

Anti-affinity snippet

yaml
Copy code
affinity:
  podAntiAffinity:
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values: [ APP_LABEL_HERE ]
          topologyKey: kubernetes.io/hostname
k8s/deployments/user-deployment.yaml

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata: { name: user-service, namespace: ecommerce, labels: { app: user-service } }
spec:
  replicas: 3
  selector: { matchLabels: { app: user-service } }
  template:
    metadata: { labels: { app: user-service } }
    spec:
      containers:
        - name: user
          image: <your-registry>/user-service:latest
          ports: [{ containerPort: 3000 }]
          env:
            - { name: DB_HOST, value: user-postgres.ecommerce.svc.cluster.local }
            - { name: DB_PORT, value: "5432" }
            - { name: DB_USER, value: "user" }
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: USER_DB_PASSWORD } }
            - { name: DB_NAME, value: "userdb" }
            - name: JWT_SECRET
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: JWT_SECRET } }
          readinessProbe: { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 5, periodSeconds: 10 }
          livenessProbe:  { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 15, periodSeconds: 20 }
      affinity: # spread across 3 worker nodes
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector: { matchExpressions: [ { key: app, operator: In, values: [ user-service ] } ] }
              topologyKey: kubernetes.io/hostname
k8s/deployments/product-deployment.yaml

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata: { name: product-service, namespace: ecommerce, labels: { app: product-service } }
spec:
  replicas: 3
  selector: { matchLabels: { app: product-service } }
  template:
    metadata: { labels: { app: product-service } }
    spec:
      containers:
        - name: product
          image: <your-registry>/product-service:latest
          ports: [{ containerPort: 3000 }]
          env:
            - { name: DB_HOST, value: product-postgres.ecommerce.svc.cluster.local }
            - { name: DB_PORT, value: "5432" }
            - { name: DB_USER, value: "product" }
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: PRODUCT_DB_PASSWORD } }
            - { name: DB_NAME, value: "productdb" }
          readinessProbe: { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 5, periodSeconds: 10 }
          livenessProbe:  { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 15, periodSeconds: 20 }
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector: { matchExpressions: [ { key: app, operator: In, values: [ product-service ] } ] }
              topologyKey: kubernetes.io/hostname
k8s/deployments/cart-deployment.yaml

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata: { name: cart-service, namespace: ecommerce, labels: { app: cart-service } }
spec:
  replicas: 3
  selector: { matchLabels: { app: cart-service } }
  template:
    metadata: { labels: { app: cart-service } }
    spec:
      containers:
        - name: cart
          image: <your-registry>/cart-service:latest
          ports: [{ containerPort: 3000 }]
          env:
            - { name: REDIS_HOST, value: cart-redis.ecommerce.svc.cluster.local }
            - { name: REDIS_PORT, value: "6379" }
            - name: JWT_SECRET
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: JWT_SECRET } }
          readinessProbe: { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 5, periodSeconds: 10 }
          livenessProbe:  { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 15, periodSeconds: 20 }
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector: { matchExpressions: [ { key: app, operator: In, values: [ cart-service ] } ] }
              topologyKey: kubernetes.io/hostname
k8s/deployments/order-deployment.yaml

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata: { name: order-service, namespace: ecommerce, labels: { app: order-service } }
spec:
  replicas: 3
  selector: { matchLabels: { app: order-service } }
  template:
    metadata: { labels: { app: order-service } }
    spec:
      containers:
        - name: order
          image: <your-registry>/order-service:latest
          ports: [{ containerPort: 3000 }]
          env:
            - { name: DB_HOST, value: order-postgres.ecommerce.svc.cluster.local }
            - { name: DB_PORT, value: "5432" }
            - { name: DB_USER, value: "order" }
            - name: DB_PASSWORD
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: ORDER_DB_PASSWORD } }
            - { name: DB_NAME, value: "orderdb" }
            - name: JWT_SECRET
              valueFrom: { secretKeyRef: { name: ecommerce-secrets, key: JWT_SECRET } }
          readinessProbe: { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 5, periodSeconds: 10 }
          livenessProbe:  { httpGet: { path: /healthz, port: 3000 }, initialDelaySeconds: 15, periodSeconds: 20 }
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector: { matchExpressions: [ { key: app, operator: In, values: [ order-service ] } ] }
              topologyKey: kubernetes.io/hostname
k8s/deployments/frontend-deployment.yaml

yaml
Copy code
apiVersion: apps/v1
kind: Deployment
metadata: { name: frontend, namespace: ecommerce, labels: { app: frontend } }
spec:
  replicas: 3
  selector: { matchLabels: { app: frontend } }
  template:
    metadata: { labels: { app: frontend } }
    spec:
      containers:
        - name: frontend
          image: <your-registry>/ecommerce-frontend:latest
          ports: [{ containerPort: 80 }]
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector: { matchExpressions: [ { key: app, operator: In, values: [ frontend ] } ] }
              topologyKey: kubernetes.io/hostname
4.4 Services (ClusterIP)
k8s/services/user-service-svc.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: user-service, namespace: ecommerce }
spec:
  selector: { app: user-service }
  ports: [{ port: 80, targetPort: 3000 }]
  type: ClusterIP
k8s/services/product-service-svc.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: product-service, namespace: ecommerce }
spec:
  selector: { app: product-service }
  ports: [{ port: 80, targetPort: 3000 }]
  type: ClusterIP
k8s/services/cart-service-svc.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: cart-service, namespace: ecommerce }
spec:
  selector: { app: cart-service }
  ports: [{ port: 80, targetPort: 3000 }]
  type: ClusterIP
k8s/services/order-service-svc.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: order-service, namespace: ecommerce }
spec:
  selector: { app: order-service }
  ports: [{ port: 80, targetPort: 3000 }]
  type: ClusterIP
k8s/services/frontend-service-svc.yaml

yaml
Copy code
apiVersion: v1
kind: Service
metadata: { name: frontend-service, namespace: ecommerce }
spec:
  selector: { app: frontend }
  ports: [{ port: 80, targetPort: 80 }]
  type: ClusterIP
4.5 Ingress (uses your domain: ganga888.online)
Requires an Ingress Controller (install in step 6).
Paths below match the frontend’s /api/... calls.

k8s/ingress.yaml

yaml
Copy code
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ecommerce-ingress
  namespace: ecommerce
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
    - host: ganga888.online
      http:
        paths:
          - path: /api/user
            pathType: Prefix
            backend: { service: { name: user-service, port: { number: 80 } } }
          - path: /api/product
            pathType: Prefix
            backend: { service: { name: product-service, port: { number: 80 } } }
          - path: /api/cart
            pathType: Prefix
            backend: { service: { name: cart-service, port: { number: 80 } } }
          - path: /api/order
            pathType: Prefix
            backend: { service: { name: order-service, port: { number: 80 } } }
          - path: /
            pathType: Prefix
            backend: { service: { name: frontend-service, port: { number: 80 } } }
TLS (optional but recommended): See step 7 to enable HTTPS via cert-manager/Let’s Encrypt.

5) Build & push Docker images
From each directory:

bash
Copy code
# user-service
cd user-service
docker build -t <your-registry>/user-service:latest .
docker push <your-registry>/user-service:latest

# product-service
cd ../product-service
docker build -t <your-registry>/product-service:latest .
docker push <your-registry>/product-service:latest

# cart-service
cd ../cart-service
docker build -t <your-registry>/cart-service:latest .
docker push <your-registry>/cart-service:latest

# order-service
cd ../order-service
docker build -t <your-registry>/order-service:latest .
docker push <your-registry>/order-service:latest

# frontend
cd ../frontend
docker build -t <your-registry>/ecommerce-frontend:latest .
docker push <your-registry>/ecommerce-frontend:latest
6) Install Ingress Controller (nginx) and deploy
bash
Copy code
# Create namespace, secrets, config
kubectl apply -f k8s/namespace.yaml
kubectl apply -f k8s/secrets.yaml
kubectl apply -f k8s/configmaps-dbinit.yaml

# Databases (wait until Running)
kubectl apply -f k8s/user-postgres-statefulset.yaml
kubectl apply -f k8s/product-postgres-statefulset.yaml
kubectl apply -f k8s/order-postgres-statefulset.yaml
kubectl apply -f k8s/redis-statefulset.yaml
kubectl -n ecommerce get pods -w

# App Deployments + Services
kubectl apply -f k8s/deployments/
kubectl apply -f k8s/services/

# Install nginx ingress controller with Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace

# Create Ingress
kubectl apply -f k8s/ingress.yaml
Get the external address of the Ingress controller’s Service:

bash
Copy code
kubectl -n ingress-nginx get svc ingress-nginx-controller -o wide
On AWS you’ll see an EXTERNAL-IP / hostname (ELB). Copy it.

Create DNS record:

In your DNS provider, create an A (or ALIAS) record for ganga888.online pointing to the above ELB hostname/IP.

Propagation may take a few minutes.

Open http://ganga888.online/ (or https if you enabled TLS in the next step).

7) (Optional) Enable HTTPS with cert-manager (Let’s Encrypt)
bash
Copy code
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager --create-namespace --set crds.enabled=true
Create a ClusterIssuer for Let’s Encrypt HTTP-01:

yaml
Copy code
# save as k8s/clusterissuer-letsencrypt.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata: { name: letsencrypt }
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef: { name: letsencrypt-account-key }
    solvers:
    - http01:
        ingress:
          class: nginx
Patch k8s/ingress.yaml to add TLS:

yaml
Copy code
metadata:
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt
spec:
  tls:
    - hosts: [ ganga888.online ]
      secretName: ganga888-online-tls
  rules:
    - host: ganga888.online
      http:
        paths:
          ...
Apply:

bash
Copy code
kubectl apply -f k8s/clusterissuer-letsencrypt.yaml
kubectl apply -f k8s/ingress.yaml
Wait for certificate:

bash
Copy code
kubectl -n ecommerce get certificate
kubectl -n ecommerce describe certificate ganga888-online-tls
Browse https://ganga888.online.

8) Quick test flow (curl)
(If needed) port-forward services locally for testing:

bash
Copy code
kubectl -n ecommerce port-forward svc/user-service 30001:80 &
kubectl -n ecommerce port-forward svc/product-service 30002:80 &
kubectl -n ecommerce port-forward svc/cart-service 30003:80 &
kubectl -n ecommerce port-forward svc/order-service 30004:80 &
Register → Login → Use token:

bash
Copy code
curl -s -X POST -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pw"}' http://ganga888.online/api/user/register

TOKEN=$(curl -s -X POST -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"pw"}' http://ganga888.online/api/user/login | jq -r .token)

curl http://ganga888.online/api/product/products
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"productId":1,"quantity":2}' http://ganga888.online/api/cart/cart/add

curl -X POST -H "Authorization: Bearer $TOKEN" http://ganga888.online/api/order/orders
9) What each component does (use-cases)
Namespace — groups all app objects so kubectl -n ecommerce ... shows your stack in one place.

Secret — keeps credentials out of images and ConfigMaps; pods consume via env vars.

ConfigMap — ships bootstrap SQL to Postgres (it auto-runs at first start).

StatefulSet — databases need stable storage & hostname; PVCs persist data across pod restarts.

PVC/StorageClass — claims real EBS volumes for Postgres/Redis data.

Deployment (replicas: 3) — runs 3 copies, rollouts updates gradually, self-heals.

podAntiAffinity — spreads the 3 replicas across 3 worker nodes (higher availability).

Service (ClusterIP) — stable virtual IP + DNS: user-service.ecommerce.svc.cluster.local.

Ingress — single DNS (ganga888.online), path routes to backend services.

Probes — only send traffic to healthy, ready pods; restarts unhealthy ones.

10) Production notes
Use AWS RDS for Postgres instead of in-cluster DBs for HA & backups.

Add cert-manager + HTTPS (step 7) and NetworkPolicies for zero-trust between namespaces.

Add HorizontalPodAutoscaler (HPA) for autoscaling based on CPU/requests.

Centralized logging/metrics: Fluent Bit, Prometheus/Grafana, OpenTelemetry tracing.

Store secrets in AWS Secrets Manager + External Secrets Operator.

11) Troubleshooting
kubectl -n ecommerce get pods -o wide — check STATUS; describe failing pods.

DB pods CrashLoop? Verify PVCs bound and passwords in ecommerce-secrets.

No external URL? kubectl -n ingress-nginx get svc — ensure ingress-nginx-controller has EXTERNAL-IP. Point ganga888.online A/ALIAS to that.

Ingress 404s? Confirm host: ganga888.online matches exactly and DNS is propagated.

12) Cleanup
bash
Copy code
kubectl delete namespace ecommerce
helm uninstall ingress-nginx -n ingress-nginx
# (if used)
helm uninstall cert-manager -n cert-manager
13) Why one DB per microservice?
Loose coupling; each service owns its schema & scaling.

Independent tech choice per service (we chose Postgres for User/Product/Order and Redis for Cart).

Safer deployments (schema changes don’t break other services).

You’re set. Build & push images, apply the manifests, install ingress, point ganga888.online to the ingress LB, and your e-commerce demo should open in the browser. Happy shipping 🚀
