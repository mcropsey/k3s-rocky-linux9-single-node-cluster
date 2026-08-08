# Single-Node k3s + Nginx Ingress on Rocky Linux 9

Clean, tested step-by-step guide.  
This version uses **Helm + hostNetwork** for the Nginx Ingress Controller (recommended for single-node labs).

---

## 1. Basic System Prep

```bash
# Update system
sudo dnf update -y

# Disable firewalld (recommended for simple lab)
sudo systemctl stop firewalld
sudo systemctl disable firewalld

# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

---

## 2. Install k3s (Traefik disabled from the start)

```bash
curl -sfL https://get.k3s.io | sh -s - --disable=traefik --write-kubeconfig-mode 644
```

---

## 3. Set up kubectl for your user

```bash
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config
chmod 600 ~/.kube/config

# Make it permanent
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
export KUBECONFIG=~/.kube/config
```

---

## 4. Verify the cluster

```bash
kubectl get nodes
kubectl get pods -A
```

You should only see CoreDNS, local-path-provisioner, and metrics-server (no Traefik).

---

## 5. Install Helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

---

## 6. Install Nginx Ingress Controller (with hostNetwork)

This is the recommended method for a single-node k3s lab.  
It makes nginx listen directly on the host’s ports 80 and 443.

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  --set controller.hostNetwork=true \
  --set controller.hostPort.enabled=true \
  --set controller.service.type=ClusterIP \
  --set controller.kind=DaemonSet
```

Wait for the controller to become ready:

```bash
kubectl get pods -n ingress-nginx -w
```

(Press `Ctrl+C` when the controller shows `1/1 Running`)

---

## 7. Verify Nginx is listening on the host

```bash
kubectl get pods -n ingress-nginx -o wide
ss -tlnp | grep -E ':80|:443'
```

You should see the controller pod using the node IP and nginx listening on `0.0.0.0:80` and `0.0.0.0:443`.

---

## 8. Quick Test (optional)

```bash
# Create a simple nginx app
kubectl create deployment demo --image=nginx
kubectl expose deployment demo --port=80

# Create an Ingress
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: demo.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: demo
            port:
              number: 80
EOF
```

Test it:

```bash
curl -H "Host: demo.local" http://<your-server-ip>/
```

---

## Useful Commands

```bash
# Check everything
kubectl get all -A
kubectl get ingress -A

# Uninstall k3s completely (if needed)
/usr/local/bin/k3s-uninstall.sh
```

---

## Notes

- Tested on Rocky Linux 9
- Uses community ingress-nginx (fine for labs)
- Traefik is disabled at install time to avoid conflicts
- **hostNetwork + DaemonSet** is the cleanest way for single-node exposure on ports 80/443
- Avoid the old static baremetal manifests if you plan to manage the controller with Helm later
