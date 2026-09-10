# adpg-ako-l7

A minimal, working example of exposing an application at **Layer 7** on a
vSphere-hosted Kubernetes cluster -- a Tanzu Kubernetes Cluster (TKC) on vSphere
8 with Tanzu, or a VKS cluster on VCF 9 -- using **NSX Advanced Load Balancer**
through the **Avi Kubernetes Operator (AKO)**, with TLS terminated by a supplied
certificate.

Four manifests in [`adpg/`](adpg/): a namespace, an nginx application, an
Ingress, and a HostRule. Hostname is `test.adpg.local` throughout.

## Validated environments

Verified against two independent stacks. The manifests are identical on both --
nothing in `adpg/` is version-specific.

**vSphere 8 with Tanzu (TKGS / TKC)** -- compatibility validated by the platform team:

| Component | Version |
|---|---|
| vCenter Server | 8.0 U3 |
| Tanzu Kubernetes Cluster (TKC) | 1.30 |
| NSX Advanced Load Balancer (Avi Controller) | 22.1.5 |
| AKO | 1.13.4 |
| Networking | Supervisor + vSphere Distributed Switch (VDS) + Avi. No NSX, no VPC. |
| IngressClass | `avi-lb` |

**VKS / VCF 9** -- where this sample was built and exercised:

| Component | Version |
|---|---|
| Kubernetes | 1.35 |
| NSX Advanced Load Balancer (Avi Controller) | 32.1.1 |
| AKO | 2.1.3 |
| Networking | Supervisor + NSX, VPC mode enabled |
| IngressClass | `avi-lb` |

### Why the same manifests work on both

The `HostRule` in this repo is `ako.vmware.com/v1beta1`. That is the **storage
version** in AKO 1.13.4 as well as 2.x, so no API change is needed between them.
(`v1alpha1` is still served on 1.13.4 for backward compatibility, but there is no
reason to use it.) Every field the sample sets -- `fqdnType: Exact`,
`enableVirtualHost`, `applicationProfile`, `tls.termination: edge`, and
`sslKeyCertificate.type` accepting both `secret` and `ref` -- is present in the
1.13.4 CRD schema.

The two stacks also differ in networking -- VDS on one, NSX with VPC mode on the
other. That difference lands entirely in the **AKO add-on configuration**, not in
these manifests. A VDS-backed deployment uses a vCenter cloud on the Avi
controller and needs neither `vpcMode` nor an NSX T1 router setting; an
NSX-backed one does. Nothing in `adpg/` refers to the underlying network, so the
same four files apply either way.

On vSphere 8 with Tanzu, AKO is installed as a **cluster add-on** on the TKC
rather than being present by default. It is not inherited from the Supervisor:
the Supervisor's own AKO serves Supervisor-level Layer 4 only, and does not
process Ingress objects belonging to a Tanzu Kubernetes Cluster. Install the
add-on first, then confirm the `avi-lb` IngressClass exists before applying
anything here.

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


## Deploy

### 1. Namespace

```sh
kubectl apply -f adpg/00-namespace.yaml
```

### 2. Certificate

This sample references a certificate **already imported on the Avi controller**,
so there is nothing to create in Kubernetes. Import it once, under
*Templates > Security > SSL/TLS Certificates > Create > Application
Certificate*, with **Type** set to `Import`:

- paste the certificate PEM into the certificate field
- paste the unencrypted private key PEM into the key field
- name the object exactly `adpg-test` -- this is the name
  [`adpg/30-hostrule.yaml`](adpg/30-hostrule.yaml) references

Then verify, on the controller, that the certificate's **SAN** covers
`test.adpg.local`. Modern browsers and Go's TLS stack ignore the CN entirely and
match on SAN only, so a certificate carrying just a correct CN still fails with
`ERR_CERT_COMMON_NAME_INVALID`.

> This check matters more than it looks. If the object's name is right but its
> SAN is wrong, the HostRule still reports `Accepted` and nothing in Kubernetes
> reports a problem -- the site simply fails in every browser. Kubernetes has no
> visibility into what the controller is actually serving.

Prefer to manage the certificate from Kubernetes instead? See
[Using a Kubernetes secret instead](#using-a-kubernetes-secret-instead).

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

As shipped, the certificate lives **only on the Avi controller**. The HostRule
in [`adpg/30-hostrule.yaml`](adpg/30-hostrule.yaml) names it:

```yaml
tls:
  sslKeyCertificate:
    type: ref          # an object on the controller, not a Kubernetes secret
    name: adpg-test    # must match the controller object name exactly
  termination: edge
```

[`adpg/20-ingress.yaml`](adpg/20-ingress.yaml) deliberately has **no `tls`
block**. It does not need one: a HostRule carrying an `sslKeyCertificate`
converts an insecure host FQDN into a secure one on its own. Verified end to
end -- with no Ingress `tls` section and no secret in the namespace, the
HostRule reported `Accepted`, the Avi child virtual service kept its certificate
attached, and the VIP terminated TLS correctly.

The trade-off: the certificate is not described by your manifests. Renewals
happen on the controller, and a hostname mismatch there is invisible from
Kubernetes. Someone has to own checking CN and SAN on the controller object, at
import and at every renewal.

### Using a Kubernetes secret instead

If you would rather the certificate be versioned alongside the manifests, with
no manual step on the controller, AKO can upload it for you.

1. Create the secret, which keeps the private key out of git:

   ```sh
   kubectl -n adpg create secret tls adpg-test \
     --cert=/path/to/test.adpg.local.crt \
     --key=/path/to/test.adpg.local.key
   ```

2. In [`adpg/30-hostrule.yaml`](adpg/30-hostrule.yaml), change
   `sslKeyCertificate.type` from `ref` to `secret`. The `name` stays the same --
   it now refers to the Kubernetes secret rather than a controller object.

3. In [`adpg/20-ingress.yaml`](adpg/20-ingress.yaml), add the `tls` block back
   (the exact YAML is in a comment at the top of that file).

Confirm what you loaded matches the hostname:

```sh
kubectl -n adpg get secret adpg-test -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -subject -ext subjectAltName
```

When both a HostRule certificate and an Ingress secret are present, **the
HostRule wins**.

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
kubectl delete -f adpg/00-namespace.yaml
```

Deleting the Ingress and HostRule removes the virtual service and pools from the
Avi controller.

The **certificate object is not removed**. AKO deletes only what it created, and
with `sslKeyCertificate.type: ref` the certificate was imported by hand, so it
stays on the controller for you to remove or reuse. (If you switched to
`type: secret`, AKO uploaded the certificate and does clean it up -- and you
should also `kubectl -n adpg delete secret adpg-test`.)
