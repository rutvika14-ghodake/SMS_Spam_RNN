# SMS Spam RNN — Full Production Deployment Guide (EKS + nginx Ingress)

Cluster name: **sms-app** | Account: **714781437992** | Region: **us-east-1**

---

## Phase 0 — Prerequisites

- [ ] AWS CLI configured with credentials that can create ECR repos, EKS clusters, IAM roles: `aws sts get-caller-identity` works.
- [ ] Docker installed and working: `docker ps` returns without error.
- [ ] `eksctl`, `kubectl`, and `helm` installed locally or on your EC2 instance.
- [ ] (Optional, for HTTPS) A domain you control, with DNS managed somewhere you can add records (e.g. Route53).

---

## Phase 1 — Build and push the Docker image

```bash
cd ~/SMS_Spam_RNN

# Build the image (uses the multi-stage Dockerfile)
docker build -t sms-spam-rnn:latest .

# Create the ECR repo (one-time)
aws ecr create-repository --repository-name sms-spam-rnn --region us-east-1

# Authenticate Docker to ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin 714781437992.dkr.ecr.us-east-1.amazonaws.com

# Tag and push
docker tag sms-spam-rnn:latest 714781437992.dkr.ecr.us-east-1.amazonaws.com/sms-spam-rnn:latest
docker push 714781437992.dkr.ecr.us-east-1.amazonaws.com/sms-spam-rnn:latest
```

Confirm the push succeeded — last line should read `latest: digest: sha256:... size: ...`.

---

## Phase 2 — Create the EKS cluster

```bash
eksctl create cluster \
  --name sms-app \
  --region us-east-1 \
  --version 1.30 \
  --nodegroup-name default-ng \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --with-oidc \
  --managed
```

Takes ~15-20 minutes. **Cost note:** this cluster runs roughly $150-165/month if left up (control plane + 2 nodes + NAT gateway) — delete it when done demoing (`eksctl delete cluster --name sms-app --region us-east-1`).

```bash
kubectl get nodes
```
Confirm 2 nodes show `Ready`.

---

## Phase 3 — Deploy the app

Apply the Deployment and the (ClusterIP) Service:

```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl get pods -w
```

Wait for both pods to show `Running` / `1/1 Ready` — each pod loads the full TensorFlow model, so this can take 20-30 seconds per pod.

---

## Phase 4 — Install the nginx Ingress Controller

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

```bash
kubectl get pods -n ingress-nginx
```
Wait for the controller pod to be `Running`.

```bash
kubectl get svc -n ingress-nginx
```
Note the `EXTERNAL-IP` (an NLB DNS name) on the `ingress-nginx-controller` service — this is your public entry point.

---

## Phase 5 — Decide: HTTP-only demo, or full HTTPS with a domain?

**Option A — Skip TLS for now (quick demo via raw NLB address)**

Apply the basic (no-TLS) Ingress — remove the `tls:` block and `host:` fields from `ingress.yaml`, or use a simple version with just a default path rule. Then:
```bash
kubectl apply -f ingress.yaml
kubectl get ingress
```
Visit `http://<the-nlb-dns-name-from-phase-4>/` once the `ADDRESS` column populates.

**Option B — Full production HTTPS with your own domain**

Continue to Phase 6.

---

## Phase 6 — Add HTTPS via cert-manager + Let's Encrypt (production path)

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
kubectl get pods -n cert-manager
```
Wait for all 3 pods (`cert-manager`, `cainjector`, `webhook`) to be `Running`.

```bash
kubectl apply -f cluster-issuer.yaml
kubectl get clusterissuer letsencrypt-prod
```
`READY` should say `True`.

**Point your domain at the nginx NLB:**
In Route53 (or your DNS provider), create a `CNAME` record:
```
sms-spam.yourdomain.com  ->  <the NLB DNS name from Phase 4>
```

**Update the Ingress with your real domain:**
Edit `ingress.yaml`, replace both instances of `sms-spam.yourdomain.com` with your actual subdomain, then:
```bash
kubectl apply -f ingress.yaml
kubectl describe certificate sms-spam-rnn-tls
```
Wait 1-2 minutes for Let's Encrypt to issue the cert via the HTTP-01 challenge — `Ready: True` when done.

```bash
kubectl get ingress
```
`ADDRESS` should match the NLB. Visit `https://sms-spam.yourdomain.com` — should load with a valid padlock.

---

## Phase 7 — Sanity checks

```bash
# Confirm pods are healthy
kubectl get pods -o wide

# Confirm the Service routes correctly (from inside the cluster)
kubectl run curl-test --rm -it --image=curlimages/curl -- curl http://sms-spam-rnn-svc

# Tail logs if something looks wrong
kubectl logs -f deployment/sms-spam-rnn
```

---

## Phase 8 — Post-deployment verification checklist

Work through these top to bottom — they check from the inside out (pod → service → ingress → external access → resilience), so if something's broken you catch it at the right layer instead of guessing.

1. **Pod health** — `kubectl get pods -o wide`. Both replicas should show `Running`, `1/1 Ready`, `RESTARTS: 0`. If a pod is `CrashLoopBackOff` or restarting, check `kubectl logs <pod-name>`.

2. **Application logs** — `kubectl logs -f deployment/sms-spam-rnn`. Both pods should show `Model loaded successfully!` and gunicorn's startup lines, no Python tracebacks.

3. **Service resolves internally** — `kubectl get svc sms-spam-rnn-svc` (should be `ClusterIP` with a valid `CLUSTER-IP`). Then test from inside the cluster:
   ```bash
   kubectl run curl-test --rm -it --image=curlimages/curl -- curl -I http://sms-spam-rnn-svc
   ```
   Should return `HTTP/1.1 200 OK`.

4. **Ingress Controller and Ingress resource** —
   ```bash
   kubectl get pods -n ingress-nginx      # controller pod Running
   kubectl get svc -n ingress-nginx       # EXTERNAL-IP populated, not <pending>
   kubectl get ingress                    # ADDRESS populated, matches the NLB name
   ```

5. **TLS certificate (if using HTTPS)** — `kubectl describe certificate sms-spam-rnn-tls`, look for `Ready: True`. If stuck on `False`, run `kubectl describe challenge` to see what's blocking Let's Encrypt's domain verification (usually DNS not propagated yet).

6. **End-to-end test from outside the cluster** — From your own browser, visit `http://<nlb-name>/` or `https://yourdomain.com/`. The classifier page should render; submit a test message and confirm you get a HAM/SPAM result with no error page.

7. **Resource usage** — `kubectl top pods` (install metrics-server first if missing: `kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml`). Confirm memory usage stays comfortably under the 2Gi limit per pod, not near the ceiling (risk of `OOMKilled`).

8. **Self-healing test (optional but good practice)** —
   ```bash
   kubectl delete pod <one-of-the-two-pod-names>
   kubectl get pods -w
   ```
   A replacement pod should reach `Running`/`1/1 Ready` within ~30-40 seconds, with the app staying reachable throughout via the other replica.

---

## Known open item (not infra)

Predictions on real text input are currently unreliable — the app's `preprocess_input()` falls back to a raw character-code encoding when no tokenizer is present, which doesn't match how the model was actually trained. See the earlier deployment doc for details. Doesn't block any of the above; worth fixing before treating this as a working demo.

## Teardown (when you're done)

```bash
kubectl delete -f ingress.yaml
kubectl delete -f service.yaml
kubectl delete -f deployment.yaml
helm uninstall ingress-nginx -n ingress-nginx
eksctl delete cluster --name sms-app --region us-east-1
```
