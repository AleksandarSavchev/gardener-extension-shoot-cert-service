---
title: Manage certificates with Gardener for public domain
level: beginner
category: Networking
scope: operator
publishdate: 2022-07-20
tags: ["task"]
description: Use the Gardener cert-management to get fully managed, publicly trusted TLS certificates
---

# Manage certificates with Gardener for public domain
## Introduction
Dealing with applications on Kubernetes which offer a secure service endpoints (e.g. HTTPS) also require you to enable a 
secured communication via SSL/TLS. With the [certificate extension](https://github.com/gardener/gardener-extension-shoot-cert-service) enabled, Gardener can manage commonly trusted X.509 certificate for your application 
endpoint. From initially requesting certificate, it also handeles their renewal in time using the free Let's Encrypt API.

**There are two senarios with which you can use the certificate extension**
- You want to use a certificate for a subdomain the shoot's default DNS (see `.spec.dns.domain` of your shoot resource, e.g. `short.ingress.shoot.project.default-domain.gardener.cloud`). If this is your case, please see [Manage certificates with Gardener for default domain](./request_default_domain_cert.md)
- You want to use a certificate for a custom domain. If this is your case, please keep reading this article.
## Prerequisites

Before you start this guide there are a few requirements you need to fulfill:

- You have an existing shoot cluster
- Your custom domain is under a [public top level domain](https://www.iana.org/domains/root/db) (e.g. `.com`)
- Your custom zone is resolvable with a public resolver via the internet (e.g. `8.8.8.8`)
- You have a custom DNS provider configured and working (see ["DNS Providers"](https://github.com/gardener/gardener-extension-shoot-dns-service/blob/master/docs/usage/dns_providers.md))

As part of the [Let's Encrypt](https://letsencrypt.org/) [ACME](https://tools.ietf.org/html/rfc8555) challenge validation process, Gardener sets a DNS TXT entry and Let's Encrypt checks if it can both resolve and authenticate it. Therefore, it's important that your DNS-entries are publicly resolvable. You can check this by querying e.g. Googles public DNS server and if it returns an entry your DNS is publicly visible:

```bash
# returns the A record for cert-example.example.com using Googles DNS server (8.8.8.8)
dig cert-example.example.com @8.8.8.8 A
```

### DNS provider
In order to issue certificates for a custom domain you need to specify a DNS provider which is permitted to create DNS records for subdomains of your requested domain in the certificate. For example, if you request a certificate for `host.example.com` your DNS provider must be capable of managing subdomains of `host.example.com`.

DNS providers are normally specified in the shoot manifest. To learn more on how to configure one, please see the [DNS provider](https://github.com/gardener/gardener-extension-shoot-dns-service/blob/master/docs/usage/dns_providers.md) documentation.

## Issue a certificate
Every X.509 certificate is represented by a Kubernetes custom resource `certificate.cert.gardener.cloud` in your cluster. A `Certificate` resource may be used to initiate a new certificate request as well as to manage its lifecycle. Gardener's certificate service regularly checks the expiration timestamp of Certificates, triggers a renewal process if necessary and replaces the existing X.509 certificate with a new one.

> Your application should be able to reload replaced certificates in a timely manner to avoid service disruptions.

Certificates can be requested via 3 resources type
- Ingress
- Service (type LoadBalancer)
- Certificate (Gardener CRD)

If either of the first 2 are used, a corresponding `Certificate` resource will created automatically.

### Using an ingress Resource
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: amazing-ingress
  annotations:
    cert.gardener.cloud/purpose: managed
    # Optional but recommended, this is going to create the DNS entry at the same time
    dns.gardener.cloud/class: garden
    dns.gardener.cloud/ttl: "600"
spec:
  tls:
  - hosts:
    # Must not exceed 64 characters.
    - amazing.example.com
    # Certificate and private key reside in this secret.
    secretName: tls-secret
  rules:
  - host: amazing.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: amazing-svc
            port:
              number: 8080
```

Replace the `hosts` and `rules[].host` value again with your own domain and adjust the remaining Ingress attributes in accordance with your deployment (e.g. the above is for an `istio` Ingress controller and forwards traffic to a `service1` on port 80).

### Using a service type LoadBalancer
```yaml
apiVersion: v1
kind: Service
metadata:
  annotations:
    cert.gardener.cloud/secretname: tls-secret
    dns.gardener.cloud/dnsnames: example.example.com
    dns.gardener.cloud/class: garden
    # Optional
    dns.gardener.cloud/ttl: "600"
    cert.gardener.cloud/commonname: "*.example.example.com"
    cert.gardener.cloud/dnsnames: ""
  name: test-service
  namespace: default
spec:
  ports:
    - name: http
      port: 80
      protocol: TCP
      targetPort: 8080
  type: LoadBalancer
```

### Using the custom Certificate resource
```yaml
apiVersion: cert.gardener.cloud/v1alpha1
kind: Certificate
metadata:
  name: cert-example
  namespace: default
spec:
  commonName: amazing.example.com
  secretRef:
    name: tls-secret
    namespace: default
  # Optionnal if using the default issuer
  issuerRef:
    name: garden
```

## Supported attributes
Here is a list of all supported annotations regarding the certificate extension:

|  Path  | Annotation | Value | Required | Description  |
|:----|:----|:----|:----|:----|
| N/A | `cert.gardener.cloud/purpose:` | `managed` | Yes when using annotations | Flag for Gardener that this specific Ingress or Service requires a certificate |
| `spec.commonName` | `cert.gardener.cloud/commonname:` | E.g. "*.demo.example.com" or <br> "special.example.com" | Certificate and Ingress : No <br/> Service: yes | Specifies for which domain the certificate request will be created. If not specified, the names from spec.tls[].hosts are used. This entry must comply with the [64 character](#Character-Restrictions) limit. |
| `spec.dnsName` | `cert.gardener.cloud/dnsnames:` | E.g. "special.example.com" | Certificate and Ingress : No <br/> Service: yes | Additional domains the certificate should be valid for (Subject Alternative Name). If not specified, the names from spec.tls[].hosts are used. Entries in this list can be longer than 64 characters. |
| `spec.secretRef.name` | `cert.gardener.cloud/secretname:` | `any-name` | Yes for certificate and Service | Specifies the secret which contains the certificate/key pair. If the secret is not available yet, it'll be created automatically as soon as the certificate has been issued. |
| `spec.issuerRef.name` | `cert.gardener.cloud/issuer:` | E.g. `gardener` | No | Specifies the issuer you want to use. Only necessary if you request certificates for [custom domains](#Custom-Domains). |
| N/A | `cert.gardener.cloud/revoked:` | `true` otherwise always false | No | Use only to revoke a certificate, see [reference](#references) for more details |

## Request a wildcard certificate
In order to avoid the creation of multiples certificates for every single endpoints, you may want to create a wildcard certificate for your shoot's default cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: amazing-ingress
  annotations:
    cert.gardener.cloud/purpose: managed
    cert.gardener.cloud/commonName: "*.example.com"
spec:
  tls:
  - hosts:
    - amazing.example.com
    secretName: tls-secret
  rules:
  - host: amazing.example.com
    http:
      paths:
      - pathType: Prefix
        path: "/"
        backend:
          service:
            name: amazing-svc
            port:
              number: 8080
```

Please note that this can also be achived by directly adding an annotation to a Service type LoadBalancer. You could also create a Certificate object with a wildcard domain.

## Using a custom Issuer
Most Gardener deployment with the certification extension enabled have a preconfigured `garden` issuer. It is also usually configured to use Let's Encrypt as the certificate provider.

If you need a custom issuer for a specific cluster, please see [Using a custom Issuer](./custom_shoot_issuer.md)

## Quotas

For security reasons there may be a default quota on the certificate requests per day set globally in the controller
registration of the shoot-cert-service. 

The default quota only applies if there is no explicit quota defined for the issuer itself with the field
`requestsPerDayQuota`, e.g.:

```yaml
kind: Shoot
...
spec:
  extensions:
  - type: shoot-cert-service
    providerConfig:
      apiVersion: service.cert.extensions.gardener.cloud/v1alpha1
      kind: CertConfig
      issuers:
        - email: your-email@example.com
          name: custom-issuer # issuer name must be specified in every custom issuer request, must not be "garden"
          server: 'https://acme-v02.api.letsencrypt.org/directory'
          requestsPerDayQuota: 10
```

## DNS Propagation
As stated before, cert-manager uses the ACME challenge protocol to authenticate that you are the DNS owner for the domain's certificate you are requesting. This works by creating a DNS TXT record in your DNS provider under `_acme-challenge.example.example.com` containing a token to compare with. The TXT record is only visible during the domain validation. Typically, the record is propagated within a few minutes. But if the record is not visible to the ACME server for any reasons, the certificate request is retried again after several minutes. This means you may have to wait up to one hour after the propagation problem has been resolved before the certificate request is retried. Take a look in the events with `kubectl describe ingress example` for troubleshooting.

## Character Restrictions
Due to the ACME protocol specification, at least one domain of the domains you request a certificate for must not exceed a character limit of 64  (CN - Common Name).

For example, the following request is invalid:

```yaml
apiVersion: cert.gardener.cloud/v1alpha1
kind: Certificate
metadata:
  name: cert-invalid
  namespace: default
spec:
  commonName: morethan64characters.ingress.shoot.project.default-domain.gardener.cloud
```

But it is valid to request a certificate for this domain if you have at least one domain which does not exceed the mentioned limit:

```yaml
apiVersion: cert.gardener.cloud/v1alpha1
kind: Certificate
metadata:
  name: cert-example
  namespace: default
spec:
  commonName: short.ingress.shoot.project.default-domain.gardener.cloud
  dnsNames:
  - morethan64characters.ingress.shoot.project.default-domain.gardener.cloud
```

## References
- [Gardener cert-management](https://github.com/gardener/cert-management)
- [Managing DNS with Gardener](https://github.com/gardener/gardener-extension-shoot-dns-service)
