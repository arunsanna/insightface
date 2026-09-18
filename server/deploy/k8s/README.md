# Kubernetes deployment (Forge / k3s)

Upstream phase-one ships Docker Compose only; this directory translates the
Compose contract to Kubernetes for the ArunLabs Forge cluster
(single node `megamind-dc`, 6x RTX 5090, local-path storage, Istio).

Contract mirrored from `../compose.cuda12.yml`:

- image `ghcr.io/deepinsight/insightface-server:0.3.1-cuda12`, port 8080
- writable mounts: `/models` (PVC, written by the install Job), `/data` (PVC,
  SQLite), `/etc/insightface` (ConfigMap), `/tmp` (memory emptyDir)
- `INSIGHTFACE_STRICT_CUDA=1`: startup fails loudly without a GPU; no CPU fallback
- models are installed by a one-shot Job (`models_cli install buffalo_l`),
  never at server startup; downloads come from the GitHub `model-zoo` release
- auth: `INSIGHTFACE_AUTH_ENABLED=true` + `INSIGHTFACE_API_KEY` from a Secret
- route: `insightface.arunlabs.com` via `istio-system/arunlabs-private-gateway`
  (biometric data — keep off the public gateway)

## Apply order

```bash
kubectl apply -f 00-namespace.yaml
kubectl apply -f 01-configmap.yaml -f 02-pvc.yaml
kubectl create secret generic insightface-api-key -n insightface \
  --from-literal=INSIGHTFACE_API_KEY='<generated-key>'   # not committed
kubectl apply -f 10-models-install-job.yaml
kubectl wait --for=condition=complete job/insightface-models-install -n insightface --timeout=300s
kubectl apply -f 20-deployment.yaml -f 30-service.yaml -f 40-virtualservice.yaml
kubectl rollout status deploy/insightface-server -n insightface
```

DNS: create the `insightface.arunlabs.com` record (private resolver) pointing
at the Istio ingress LB before the route is usable.
