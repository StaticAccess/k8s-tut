# Istio VirtualService Rewrite: Why We Use It

In our Istio setup, we expose multiple applications (e.g., Prometheus, Grafana, ProductPage) via a single Istio Ingress Gateway using different URI prefixes like:

- `/prometheus`
- `/grafana`
- `/productpage`

## 🧠 Why `rewrite.uri` is Needed

Some applications (like Prometheus, Grafana, Kiali, etc.) are **hardcoded to serve their web UI from the root path (`/`) only**. These apps break if you access them using a path like `/prometheus` or `/grafana`.

---

### 🔁 How Traffic Routing Works

When you access:
```
http://localhost:8080/prometheus
```

Istio performs these steps:

1. **Match** the URI path:
   - `/prometheus`

2. **Route** to the correct internal service:
   - `prometheus.istio-system.svc.cluster.local`

3. **Rewrite** the path:
   - Original path `/prometheus` is rewritten to `/`

4. **Prometheus receives**:
   - `GET /`

✅ This works perfectly because Prometheus is coded to serve its dashboard at `/`.

---

### ❌ What Happens Without Rewrite

If we skip the rewrite, Prometheus receives:
```
GET /prometheus
```

But Prometheus is **not coded to handle `/prometheus`**, so it responds with:

- 404 Not Found
- Broken CSS/JS
- Blank page

---

### ✅ When Rewrite is NOT Needed (e.g., ProductPage)

Some apps like `ProductPage` are custom-built and **do expect their full path**, such as `/productpage`.

So in this case, we should NOT rewrite the URI, because:

- The app is coded to respond to `/productpage`
- Rewriting to `/` would break routing

---

## 🧪 Summary

| App                      | URI Prefix     | Rewrite Needed? | Final URI Sent | Why?                                |
|---------------------------|----------------|------------------|----------------|-------------------------------------|
| Prometheus                | `/prometheus`  | ✅ Yes           | `/`            | Serves UI only from root            |
| Grafana                   | `/grafana`     | ✅ Yes           | `/`            | Serves UI only from root            |
| Kiali                     | `/kiali`       | ✅ Yes           | `/`            | Serves UI only from root            |
| ProductPage (custom app)  | `/productpage` | ❌ No            | `/productpage` | App is coded to serve that path     |

---

## 🛠️ Istio Code Example

```yaml
http:
  - match:
      - uri:
          prefix: /prometheus
    rewrite:
      uri: /
    route:
      - destination:
          host: prometheus.istio-system.svc.cluster.local
          port:
            number: 9090
```

Here:
- Incoming traffic to `/prometheus` is rewritten to `/`
- Then it is sent to `prometheus.istio-system.svc.cluster.local:9090`

```
Browser Request   →  http://localhost:8080/prometheus
Istio Match       →  /prometheus
Istio Rewrite     →  /
Istio Routes To   →  prometheus.istio-system.svc.cluster.local:9090
Prometheus Sees   →  GET /
```

This design allows us to expose many services on a single gateway, with clean and intuitive path prefixes on the frontend, while maintaining compatibility with backend expectations.