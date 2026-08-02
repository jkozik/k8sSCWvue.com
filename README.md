# k8sSCWvue.com

Setup sancapweather.com in a Kubernetes cluster as deployment named scwvuecom; expose as
service scwvuecom; connect to the internet as sancapweather.com through an HTTPRoute pointing
to the Envoy Gateway. A Vantage Vue weather station in Sanibel Island (San Cap) feeds a Davis
Envoy receiver; Cumulus software on a Windows 10 VM collects the data and generates station
graphs; Saratoga weather software (this deployment) renders them.

Source image [InstallSCWvue.com](https://github.com/jkozik/InstallSCWvue.com)

## Directory structure

```
k8sSCWvue.com/
├── scwvuecom-pv.yml          # PersistentVolume — NFS 192.168.100.153:/home/nfs/weather-stations/sancap/public_html
├── scwvuecom-pvc.yml         # PersistentVolumeClaim (storageClass: nfs-weather, 5Gi ROX)
├── scwvuecom-deploy.yml      # Deployment (1 replica, jkozik/scwvue.com:v2.5a)
├── scwvuecom-svc.yml         # NodePort service
├── scwvuecom-httproute.yaml  # HTTPRoute — sancapweather.com via Envoy Gateway port 30458
├── README.md                 # This file
└── old/
    ├── scwvuecom-ingress.yml # Retired nginx Ingress (kept for reference)
    ├── scwcom-deploy.yml     # Earlier scwcom variant (kept for reference)
    ├── scwcom-cronjob.yml    # Earlier cronjob (kept for reference)
    └── k9s_linux_amd64.deb  # k9s binary (kept for reference)
```

## Prerequisites

- Envoy Gateway running with `weather-gateway` on NodePort 30458
- NFS server 192.168.100.153 exporting `/home/nfs/weather-stations/sancap/public_html`
- `nfs-common` installed on all cluster nodes

## Deploy

Apply in order:

```bash
cd ~/projects/k8sSCWvue.com

# 1. Storage
kubectl apply -f scwvuecom-pv.yml
kubectl apply -f scwvuecom-pvc.yml

# Verify PVC is Bound before continuing
kubectl get pv,pvc -l app=scwvuecom

# 2. Application
kubectl apply -f scwvuecom-svc.yml
kubectl apply -f scwvuecom-deploy.yml

# 3. Routing
kubectl apply -f scwvuecom-httproute.yaml
```

## Verify

```bash
kubectl get deployment,service,pod,httproute,pv,pvc -l app=scwvuecom

# Expected:
# deployment.apps/scwvuecom   1/1
# service/scwvuecom           NodePort  80:<nodeport>/TCP
# pod/scwvuecom-<hash>        1/1 Running
# httproute/scwvuecom-route   sancapweather.com, www.sancapweather.com
# pv/scwvuecom-persistent-storage   Bound
# pvc/scwvuecom-persistent-storage  Bound
```

Test via NodePort directly:
```bash
curl http://<node-ip>:<nodeport>/ | head -5
```

Test via Envoy Gateway (matches production path):
```bash
curl -H "Host: sancapweather.com" http://<node-ip>:30458/ | head -5
# Should return Sanibel Island weather HTML
```

Verify live NFS weather data is visible inside the pod:
```bash
kubectl exec -it deploy/scwvuecom -- ls /var/www/html/mount
# Expected: cumulus  saratoga  webcam
```

## Cloudflare tunnel

Point the `sancapweather.com` and `www.sancapweather.com` public hostnames in the Cloudflare
Zero Trust tunnel to:
```
http://<node-ip>:30458
```

## NFS share (reference)

The NFS export is on 192.168.100.153 (dell3):
```
/home/nfs/weather-stations/sancap/public_html  192.168.100.0/24(ro,sync,no_root_squash)
```

Mounted read-only at `/var/www/html/mount` inside the container.

## Build image / push to Docker Hub

```bash
docker login
docker tag jkozik/scwvue.com jkozik/scwvue.com:v2.5a
docker push jkozik/scwvue.com:v2.5a
```

## Ingress → HTTPRoute migration

Ingress (nginx) has been deprecated on this cluster. Traffic is managed via the Kubernetes
Gateway API implemented by Envoy Gateway. The old ingress yaml is preserved in `old/` for
reference only — do not apply it on the new cluster.
