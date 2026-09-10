# adpg-ako-l7

Exposing an application at **Layer 7** on a vSphere-hosted Kubernetes cluster
using **NSX Advanced Load Balancer** via the **Avi Kubernetes Operator (AKO)**,
with TLS terminated by a certificate held on the Avi controller.

Four manifests in [`adpg/`](adpg/): namespace, nginx app, Ingress, HostRule.
Hostname is `test.adpg.local` throughout.

## Validated environments

| | vSphere 8 with Tanzu | VKS / VCF 9 |
|---|---|---|
| vCenter | 8.0 U3 | — |
| Cluster | TKC 1.30 | Kubernetes 1.35 |
| Avi Controller | 22.1.5 | 32.1.1 |
| AKO | 1.13.4 | 2.1.3 |
| Networking | Supervisor + VDS | Supervisor + NSX, VPC mode |
| IngressClass | `avi-lb` | `avi-lb` |

The manifests are identical on both. `HostRule` is `ako.vmware.com/v1beta1`,
which is the storage version in 1.13.4 as well as 2.x, and every field used
exists in the 1.13.4 schema. The VDS/NSX difference affects only the AKO add-on
configuration, not these files.

## Prerequisites

**AKO must already be installed.** This project does not install it, and it is
not inherited from the Supervisor — the Supervisor's AKO serves Supervisor-level
L4 only and does not process a guest cluster's Ingress objects. On vSphere 8
with Tanzu it is a TKC add-on.

```sh
kubectl -n avi-system get pods      # ako-0 Running
kubectl get ingressclass            # avi-lb present
```

If `avi-lb` is missing, stop — the Ingress will never be given an address.

## Deploy

**1. Import the certificate on the Avi controller.** Under *Templates >
Security > SSL/TLS Certificates > Create > Application Certificate*, Type
`Import`. Paste the certificate and the unencrypted key. Name it exactly
`adpg-test`, which is what [`adpg/30-hostrule.yaml`](adpg/30-hostrule.yaml)
references.

> Check the certificate's **SAN** covers `test.adpg.local`. Browsers match on
> SAN and ignore the CN. A mismatch here is invisible from Kubernetes — the
> HostRule still reports `Accepted` while the site fails in every browser.

**2. Apply the manifests.**

```sh
kubectl apply -f adpg/00-namespace.yaml
kubectl apply -f adpg/10-app.yaml
kubectl -n adpg rollout status deploy/adpg-nginx
kubectl apply -f adpg/20-ingress.yaml
kubectl apply -f adpg/30-hostrule.yaml
```

The Ingress carries no `tls` block on purpose. A HostRule with an
`sslKeyCertificate` converts an insecure host FQDN to a secure one by itself.

## Verify

```sh
kubectl -n adpg get ingress adpg-nginx     # ADDRESS becomes the Avi VIP
kubectl -n adpg get hostrule adpg-test     # STATUS Accepted

VIP=$(kubectl -n adpg get ingress adpg-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -sk --resolve test.adpg.local:443:$VIP https://test.adpg.local/ | grep 'ADPG sample app'
openssl s_client -connect $VIP:443 -servername test.adpg.local </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -dates
```

`test.adpg.local` needs a DNS A record pointing at the VIP, or `--resolve`.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Ingress `ADDRESS` empty | AKO not running, or `ingressClassName` does not match the installed IngressClass. |
| HostRule `Rejected` | No certificate object named `adpg-test` on the controller. `kubectl -n adpg describe hostrule adpg-test`. |
| Connects, then empty reply | Avi pool has no healthy servers — usually the Service type does not match AKO's `serviceType` mode, or the Service Engines cannot reach the nodes on the nodePort. |
| Browser cert warning | SAN on the controller object does not cover the hostname. |
| Pods rejected by admission | PodSecurity `restricted`. The manifest complies; a substituted image may not. |

```sh
kubectl -n avi-system logs ako-0 -c ako --tail=100
```

## Cleanup

```sh
kubectl delete -f adpg/30-hostrule.yaml -f adpg/20-ingress.yaml -f adpg/10-app.yaml
kubectl delete -f adpg/00-namespace.yaml
```

Removes the virtual service and pools from Avi. The **certificate object stays**
— AKO deletes only what it created, and this one was imported by hand.
