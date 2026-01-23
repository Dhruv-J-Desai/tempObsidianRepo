Nice — with AKS access you can debug this properly. Here’s a **straightforward checklist** that will take you from “502” → “exact root cause” quickly.

## 0) Make sure you’re on the right cluster/context

```bash
kubectl config current-context
kubectl cluster-info
```

## 1) Confirm the VirtualServer state + why it’s invalid

(Use the namespace Ramesh used — looks like `tdvip-tdvip-pbg` / `tdvip-tdvip-bbg` depending on env.)

```bash
kubectl -n tdvip-tdvip-pbg get virtualserver
kubectl -n tdvip-tdvip-pbg describe virtualserver vs-pde-vpda-services
```

✅ The **Events** section at the bottom is the goldmine. It will say things like:

- service not found
    
- port not defined on service
    
- invalid route
    
- TLS secret missing
    
- upstream has no endpoints
    

## 2) Verify the host matches the domain you’re testing

```bash
kubectl -n tdvip-tdvip-pbg get virtualserver vs-pde-vpda-services -o yaml | egrep -n "host:|name:|tls:|secret:"
```

Make sure `spec.host` is exactly the new DNS (example: `pde.tdvip.dev.azure.td.com`).

## 3) Check upstream service + endpoints (most common 502 cause)

From your earlier screenshot, this was likely the issue (port mismatch / no endpoints).

```bash
kubectl -n tdvip-tdvip-pbg get svc config-service -o yaml | egrep -n "name:|port:|targetPort:"
kubectl -n tdvip-tdvip-pbg get endpoints config-service -o wide
kubectl -n tdvip-tdvip-pbg get pods -l app=config-service -o wide
```

What you’re looking for:

- **Endpoints list is empty** → service selector/pods not ready → 502
    
- **Endpoints show port 8560 but VirtualServer upstream uses 8080** → 502
    

## 4) Check NGINX Ingress Controller logs for the exact rejection

First find the ingress controller namespace/pods (often `ingress-nginx` or similar):

```bash
kubectl get ns | egrep -i "ingress|nginx"
kubectl -n ingress-nginx get pods
kubectl -n ingress-nginx logs deploy/ingress-nginx-controller --tail=200
```

(If the deploy name differs, use the pod name.)  
These logs will often say exactly why a VS is “Invalid”.

## 5) Validate related resources (TLS + policies)

### TLS secret exists?

```bash
kubectl -n tdvip-tdvip-pbg get secret | grep -i pde-vpda-services-pathbased-vs
```

### NGINX policies exist?

```bash
kubectl -n tdvip-tdvip-pbg get nginxpolicy
```

## 6) Quick “does it work” test from inside the cluster

This helps separate “NGINX routing” vs “backend broken”.

```bash
kubectl -n tdvip-tdvip-pbg run curltest --rm -it --image=curlimages/curl -- \
  curl -sv http://config-service:8560/health
```

(Adjust port/path to what your service actually serves.)

---

# Fastest path to root cause

If you do only **two commands**, do these and paste output back (you can redact IDs):  
1)

```bash
kubectl -n tdvip-tdvip-pbg describe virtualserver vs-pde-vpda-services
```

```bash
kubectl -n tdvip-tdvip-pbg get endpoints config-service -o wide
```

With those two, I can tell you exactly whether it’s:

- port mismatch
    
- no endpoints
    
- host mismatch
    
- TLS secret/policy issue
    
- invalid route config