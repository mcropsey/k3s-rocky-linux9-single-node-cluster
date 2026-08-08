# Single-Node k3s + Nginx Ingress on Rocky Linux 9

Clean, tested step-by-step guide based on a real Rocky Linux 9 setup.

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

## 5. Install Nginx Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.12.1/deploy/static/provider/baremetal/deploy.yaml
```

Wait for it to become ready:

```bash
kubectl get pods -n ingress-nginx -w
```

(Press `Ctrl+C` when the controller is `Running`)

---

## 6. Check the Ingress service

```bash
kubectl get svc -n ingress-nginx
```

Note the **NodePort** numbers (example: `80:30905/TCP` and `443:30654/TCP`).

---

## Quick Test App (optional)

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
  - http:
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

Access it with:

```bash
curl http://<your-server-ip>:30905
```

(Replace `30905` with the actual NodePort from step 6)

---

## Useful Commands

```bash
# Check everything
kubectl get all -A

# Uninstall k3s completely (if needed)
/usr/local/bin/k3s-uninstall.sh
```

---

**Notes**
- Tested on Rocky Linux 9
- Uses community ingress-nginx (retired upstream in March 2026 – fine for labs)
- Traefik is disabled at install time to avoid cleanup issues
