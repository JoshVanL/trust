<p align="center">
  <img src="https://raw.githubusercontent.com/cert-manager/cert-manager/d7a3a3976785bd5717d9d06b115878feaf257597/logo/logo.png" height="241" width="250" alt="cert-manager project logo" />
</p>
<p align="center">
  <a href="https://godoc.org/github.com/cert-manager/trust"><img src="https://godoc.org/github.com/cert-manager/trust?status.svg" alt="cert-manager/trust godoc"></a>
  <a href="https://goreportcard.com/report/github.com/cert-manager/trust"><img alt="Go Report Card" src="https://goreportcard.com/badge/github.com/cert-manager/trust" /></a>
  <a href="https://artifacthub.io/packages/search?repo=cert-manager"><img alt="Artifact Hub" src="https://img.shields.io/endpoint?url=https://artifacthub.io/badge/repository/cert-manager" /></a>
</p>

# trust

trust is an operator for distributing trust bundles across a Kubernetes cluster.
trust is designed to complement
[cert-manager](https://github.com/cert-manager/cert-manager) by enabling services to
trust X.509 certificates signed by Issuers, as well as external CAs which may
not be known to cert-manager at all.

## Usage

trust ships with a single cluster scoped `Bundle` resource. A Bundle represents
a set of data that should be distributed and made available across the cluster.
There are no constraints on what data can be distributed.

The Bundle gathers and appends trust data from a number of `sources` located in
the trust namespace (where the trust controller is deployed), and syncs them to
a `target` in every namespace.

A typical Bundle looks like the following:

```yaml
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: my-org.com
spec:
  sources:
  # A Secret in the trust namespace created via a cert-manager Certificate
  - secret:
      name: "my-db-tls"
      key: "ca.crt"
  # A ConfigMap in the trust namespace
  - configMap:
      name: "my-org.net"
      key: "root-certs.pem"
  # An In Line
  - inLine: |
      # my-org.com CA
      -----BEGIN CERTIFICATE-----
      MIIC5zCCAc+gAwIBAgIBADANBgkqhkiG9w0BAQsFADAVMRMwEQYDVQQDEwprdWJl
      ....
      0V3NCaQrXoh+3xrXgX/vMdijYLUSo/YPEWmo
      -----END CERTIFICATE-----
  target:
    # Data synced to the ConfigMap `my-org.com` at the key `root-certs.pem` in
    # every namespace.
    configMap:
      key: "root-certs.pem"
```

Bundle currently supports the source types `configMap`, `secret` and `inLine`,
and target type `configMap`.

---

## Installation

First, install [cert-manager](https://cert-manager.io/docs/installation/) to the
cluster, and then the trust operator. It is advised to run the trust operator in
the `cert-manager` namespace.

```bash
$ helm repo add jetstack https://charts.jetstack.io --force-update
$ helm upgrade -i -n cert-manager cert-manager jetstack/cert-manager --set installCRDs=true --wait --create-namespace
$ helm upgrade -i -n cert-manager cert-manager-trust jetstack/cert-manager-trust --wait
```

#### Quick Start Example

```bash
$ kubectl create -n cert-manager configmap source-1 --from-literal=cm-key=123
$ kubectl create -n cert-manager secret generic source-2 --from-literal=sec-key=ABC
$ kubectl apply -f - <<EOF
apiVersion: trust.cert-manager.io/v1alpha1
kind: Bundle
metadata:
  name: example-bundle
spec:
  sources:
  - configMap:
      name: "source-1"
      key: "cm-key"
  - secret:
      name: "source-2"
      key: "sec-key"
  - inLine: |
      hello world!
  target:
    configMap:
      key: "target-key"
EOF
```

```bash
$ kubectl get bundle
NAME             TARGET       SYNCED   REASON   AGE
example-bundle   target-key   True     Synced   5s
```

```bash
$ kubectl get cm -A --field-selector=metadata.name=example-bundle
NAMESPACE            NAME             DATA   AGE
cert-manager         example-bundle   1      2m18s
default              example-bundle   1      2m18s
kube-node-lease      example-bundle   1      2m18s
kube-public          example-bundle   1      2m18s
kube-system          example-bundle   1      2m18s
local-path-storage   example-bundle   1      2m18s
```

```bash
$ kubectl get cm -n kube-system example-bundle -o jsonpath="{.data['target-key']}"
123
ABC
hello world!
```
