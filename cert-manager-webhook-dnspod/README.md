# DNSPod Webhook for Cert Manager

> A fork of [qqshfox/cert-manager-webhook-dnspod](https://github.com/qqshfox/cert-manager-webhook-dnspod), and is updated to cert-manager >= 0.12.0

This is a webhook solver for [DNSPod](https://www.dnspod.cn/).

## Prerequisites

* [cert-manager](https://github.com/jetstack/cert-manager): *tested with 0.12.0
    - [Installing on Kubernetes](https://docs.cert-manager.io/en/release-0.8/getting-started/install/kubernetes.html)

## Installation

Using Helm 3:

```bash
$ helm install cert-manager-webhook-dnspod ./deploy/example-webhook
```

### Prepare for DNSPod

1. Generate API ID and API Token from DNSPod (https://support.dnspod.cn/Kb/showarticle/tsid/227/)
2. Create secret to store the API Token

```bash
$ kubectl --namespace cert-manager create secret generic \
    dnspod-credentials --from-literal=api-token='<DNSPOD_API_TOKEN>'
```

## ClusterIssuer

```yaml
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # The ACME server URL
    server: https://acme-v02.api.letsencrypt.org/directory

    # Email address used for ACME registration
    email: <your email> # REPLACE THIS WITH YOUR EMAIL!!!

    # Name of a secret used to store the ACME account private key
    privateKeySecretRef:
      name: letsencrypt-prod

    solvers:
    - dns01:
        webhook:
          groupName: <your group> # REPLACE THIS TO YOUR GROUP
          solverName: dnspod
          config:
            apiID: <your dnspod api id> # REPLACE WITH API ID FROM DNSPOD!!!
            apiTokenSecretRef:
              key: api-token
              name: dnspod-credentials
```

## Certificate

```yaml
apiVersion: cert-manager.io/v1alpha2
kind: Certificate
metadata:
  # you could replace this name to your own
  name: wildcard-yourdomain-com # for *.yourdomain.com
spec:
  secretName: wildcard-yourdomain-com-tls
  renewBefore: 240h
  dnsNames:
    - '*.yourdomain.com'
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
```

## Ingress

A common use-case for cert-manager is requesting TLS signed certificates to secure your ingress resources. This can be done by simply adding annotations to your Ingress resources and cert-manager will facilitate creating the Certificate resource for you. A small sub-component of cert-manager, ingress-shim, is responsible for this.

For details, see [here](https://cert-manager.io/docs/usage/ingress/)

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - '*.yourdomain.com'
    secretName: wildcard-yourdomain-com-tls
  rules:
  - host: demo.yourdomain.com
    http:
      paths:
      - path: /
        backend:
          serviceName: backend-service
          servicePort: 80
```

## Development

All DNS providers **must** run the DNS01 provider conformance testing suite,
else they will have undetermined behaviour when used with cert-manager.

**It is essential that you configure and run the test suite when creating a
DNS01 webhook.**

An example Go test file has been provided in [main_test.go]().

Before you can run the test suite, you need to download the test binaries:

```bash
$ mkdir __main__
$ wget -O- https://storage.googleapis.com/kubebuilder-tools/kubebuilder-tools-1.14.1-darwin-amd64.tar.gz | tar x -
$ mv kubebuilder __main__/hack
```

Then modify `testdata/my-custom-solver/config.json` to setup the configs.

Now you can run the test suite with:

```bash
$ TEST_ZONE_NAME=example.com go test .
```
