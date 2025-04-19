# Setting Up a Load Balancer in `kind` with MetalLB

> Use MetalLB to enable `LoadBalancer`-type services in your local `kind` Kubernetes cluster.

---

## 🧾 Prerequisites

- You have a working `kind` cluster.
- `kubectl` is configured to talk to it.
- Your cluster nodes are using a **local network** that supports Layer 2 (e.g., 192.168.0.0/24).

---

## 1️⃣ Enable Strict ARP Mode (for IPVS only)

If you're using `kube-proxy` in **IPVS mode** (common in custom clusters), Kubernetes v1.14.2+ requires `strictARP: true`.

### Option A: Manually edit the config

```bash
kubectl edit configmap -n kube-system kube-proxy


Set the following under the `ipvs` section:

```yaml
ipvs:
  strictARP: true
```

Or patch it non-interactively:

```bash
# See the diff first
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/strictARP: false/strictARP: true/" | \
  kubectl diff -f - -n kube-system

# Apply the change
kubectl get configmap kube-proxy -n kube-system -o yaml | \
  sed -e "s/strictARP: false/strictARP: true/" | \
  kubectl apply -f - -n kube-system
```

---

## 📦 2. Install MetalLB

Apply the official manifest:

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.8/config/manifests/metallb-native.yaml
```

---

## 🌐 3. Create an IP Address Pool

Example IP range (adjust to fit your local network):

```bash
cat <<EOF > ip-address-pool.yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.0.230-192.168.0.239
EOF

kubectl apply -f ip-address-pool.yaml
```

### ❗ Troubleshooting

If you get a webhook error like:

```
Internal error occurred: failed calling webhook ... connection refused
```

Restart the MetalLB controller:

```bash
kubectl get pods -n metallb-system
kubectl delete pod <controller-pod-name> -n metallb-system
```

---

## 📢 4. Create a Layer 2 Advertisement

Tell MetalLB to advertise the IPs:

```bash
cat <<EOF > advertise.yaml
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: example
  namespace: metallb-system
spec:
  ipAddressPools:
  - first-pool
EOF

kubectl apply -f advertise.yaml
```

---

## 🧪 5. Test It — Deploy a Simple Echo Server

```bash
cat <<EOF > simple.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: policy-local
  labels:
    app: MyLocalApp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: MyLocalApp
  template:
    metadata:
      labels:
        app: MyLocalApp
    spec:
      containers:
      - name: agnhost
        image: registry.k8s.io/e2e-test-images/agnhost:2.40
        args:
          - netexec
          - --http-port=8080
          - --udp-port=8080
        ports:
        - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: lb-service-local
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: MyLocalApp
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF

kubectl apply -f simple.yaml
```

---

## ✅ Result

Once everything is up, you can run:

```bash
kubectl get svc lb-service-local
```

You should see a `EXTERNAL-IP` from your defined MetalLB pool (`192.168.0.230-239`) assigned to the service.

---

## 📚 References

- [MetalLB Installation Guide](https://metallb.universe.tf/installation/)
- [Kind Networking](https://kind.sigs.k8s.io/docs/user/loadbalancer/)
```

---

Let me know if you want this blog styled for a specific platform (e.g., Dev.to, Hugo, Jekyll), or exported as `.md`.
