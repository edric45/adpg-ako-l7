# adpg-ako-l7

A minimal, working example of exposing an application at **Layer 7** on a VMware
Kubernetes Service (VKS) cluster using **NSX Advanced Load Balancer** through the
**Avi Kubernetes Operator (AKO)**, with TLS terminated by a supplied certificate.

Four manifests in [`adpg/`](adpg/): a namespace, an nginx application, an
Ingress, and a HostRule. Hostname is `test.adpg.local` throughout.

## Prerequisites

**AKO must already be installed and healthy in the cluster.** This project does
not install it. Verify before you start:

```sh
kubectl -n avi-system get pods                 # ako-0 should be Running
kubectl get ingressclass                       # avi-lb should be listed
kubectl get crd | grep hostrules.ako.vmware.com
```

If `avi-lb` is missing, stop -- the Ingress will be created but never given an
address, because nothing is watching it.

You also need a TLS certificate and key for `test.adpg.local`, in PEM, with the
key unencrypted. Substitute your own hostname throughout if you are not using
`test.adpg.local`; it appears in `20-ingress.yaml` and `30-hostrule.yaml`.

### Know which mode AKO is running in

```sh
kubectl -n avi-system get cm avi-k8s-config -o jsonpath='{.data.serviceType}'
```

If this says `NodePort`, the Service in `10-app.yaml` must be `type: NodePort`,
which is how it ships. See the note in that file -- a ClusterIP Service in
NodePort mode produces an empty Avi pool and a virtual service that accepts
connections and returns nothing.

## Deploy

### 1. Namespace

```sh
kubectl apply -f adpg/00-namespace.yaml
```

### 2. Certificate

The certificate goes in as a standard Kubernetes TLS secret named `adpg-test`.
Created by command rather than a manifest, so the private key never enters git:

```sh
kubectl -n adpg create secret tls adpg-test \
  --cert=/path/to/test.adpg.local.crt \
  --key=/path/to/test.adpg.local.key
```

Confirm what you loaded actually matches the hostname you are about to serve:

```sh
kubectl -n adpg get secret adpg-test -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -subject -ext subjectAltName
```

The **SAN** must list `test.adpg.local`. Modern browsers and Go's TLS stack
ignore the CN entirely and match on SAN only, so a certificate carrying just a
correct CN still fails with `ERR_CERT_COMMON_NAME_INVALID`.

### 3. Application

```sh
kubectl apply -f adpg/10-app.yaml
kubectl -n adpg rollout status deploy/adpg-nginx
```

### 4. Ingress and HostRule

```sh
kubectl apply -f adpg/20-ingress.yaml
kubectl apply -f adpg/30-hostrule.yaml
```

## How the certificate is wired

The certificate is referenced in two places, and it is worth understanding which
one actually takes effect.

| | Where it points | Who uploads to Avi |
|---|---|---|
| `20-ingress.yaml` `spec.tls.secretName` | Kubernetes secret `adpg-test` | AKO |
| `30-hostrule.yaml` `sslKeyCertificate` | see below | depends on `type` |

**The HostRule wins.** When a HostRule specifies an `sslKeyCertificate`, it
overrides the Ingress `tls` block. The Ingress entry is kept as a fallback so
TLS still terminates if the HostRule is ever rejected.

As shipped, the HostRule uses `type: secret`, pointing at the same `adpg-test`
Kubernetes secret. AKO uploads it to the controller. Nothing is done by hand and
the certificate is versioned alongside the manifests.

The alternative is `type: ref`, which names a certificate object that must
already exist on the Avi controller under exactly that name, imported manually.
Use it when certificates are managed centrally rather than per-application. Full
commentary and the exact YAML are in [`adpg/30-hostrule.yaml`](adpg/30-hostrule.yaml).

> If you switch to `type: ref`, check the CN **and** SAN on the controller
> object itself. A mismatch there is invisible from Kubernetes -- the HostRule
> still reports `Accepted`, and the site still fails in every browser.

## Verify

```sh
kubectl -n adpg get ingress adpg-nginx     # ADDRESS should become the Avi VIP
kubectl -n adpg get hostrule adpg-test     # STATUS should be Accepted
```

Then, against the VIP:

```sh
VIP=$(kubectl -n adpg get ingress adpg-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# HTTP should redirect to HTTPS
curl -sI --resolve test.adpg.local:80:$VIP http://test.adpg.local/ | head -1

# HTTPS should return the page
curl -sk --resolve test.adpg.local:443:$VIP https://test.adpg.local/ | grep 'ADPG sample app'

# and present your certificate
openssl s_client -connect $VIP:443 -servername test.adpg.local </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

`test.adpg.local` will not resolve unless you add a DNS record, hence
`--resolve`. For browser access, point an A record at the VIP or add a hosts
entry. Because the certificate is self-signed in the sample, clients warn on
connect until it is added to their trust store.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Ingress `ADDRESS` stays empty | AKO not installed, not running, or `ingressClassName` does not match the installed IngressClass. |
| HostRule `Rejected` | With `type: ref`, the named certificate object does not exist on the controller. `kubectl -n adpg describe hostrule adpg-test` gives the reason. |
| Connects, then empty reply | Avi pool has no healthy servers. Check the pool in the Avi UI. Most often the Service type does not match AKO's `serviceType` mode, or the Service Engines cannot reach the cluster nodes on the nodePort. |
| Browser cert warning / name mismatch | The certificate's SAN does not cover the hostname. With `type: ref`, inspect the object on the controller, not the Kubernetes secret. |
| Pods rejected by admission | PodSecurity `restricted`. The manifest is already compliant; a substituted image may not be. |

Useful logs:

```sh
kubectl -n avi-system logs ako-0 -c ako --tail=100
kubectl -n adpg describe ingress adpg-nginx
```

## Cleanup

```sh
kubectl delete -f adpg/30-hostrule.yaml -f adpg/20-ingress.yaml -f adpg/10-app.yaml
kubectl -n adpg delete secret adpg-test
kubectl delete -f adpg/00-namespace.yaml
```

Deleting the Ingress and HostRule removes the virtual service, pools and
uploaded certificate from the Avi controller.
