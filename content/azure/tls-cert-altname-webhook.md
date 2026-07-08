+++
title = "The Webhook Died During the Handshake: Debugging ERR_TLS_CERT_ALTNAME_INVALID"
date = 2026-06-22

[taxonomies]
categories = ["post"]
tags = ["tls", "ssl", "certificate", "webhook", "dns", "debugging"]
+++

A Node.js service was forwarding a webhook to a backend API and emitted this error:

```
[onRaw] webhook forwarding error: {
  name: 'ApiError',
  title: 'Webhook forwarding failed',
  status: 502,
  message: "Hostname/IP does not match certificate's altnames:
            Host: example.com. is not in the cert's altnames:
            DNS:*.azurewebsites.net, DNS:*.scm.azurewebsites.net, ...",
  code: 'ERR_TLS_CERT_ALTNAME_INVALID'
}
```

The payload itself was fine. This was not a data problem. So why did it become a 502?

## Cause: The domain is not in the SAN

The forwarder opened an HTTPS connection to `example.com`. During the TLS handshake, the server returned a certificate whose **SAN(Subject Alternative Names)** list did not include `example.com`. In the error above, the SAN only contains names like `*.azurewebsites.net`. This is not Azure-specific. On any platform or server, if you only *route* a custom domain to the service but do not *bind* a certificate that covers that domain, you can get the same error.

A TLS client must check: "Is the hostname I am connecting to present in this certificate?" If not, it terminates the connection with `ERR_TLS_CERT_ALTNAME_INVALID`. The forwarder wrapped that failure as a 502.

**One-sentence summary:**
> Traffic was configured to *route* to `example.com`, but no TLS certificate covering `example.com` was *bound* to the server. The server fell back to a default certificate for another platform/server name, and the client rejected it because it did not match the hostname it connected to.

## Core idea: DNS routing is not TLS identity

If you mix up these two layers, you keep getting stuck.

| Layer | Question it answers | Mechanism |
|--------|----------|---------|
| **DNS / routing** | "Where should this traffic go?" | A / CNAME records |
| **TLS / identity** | "Can this server prove it is `example.com`?" | Certificate containing `example.com` |

> **Traffic reaching a domain does not mean the server has a certificate for that domain.** The certificate must be issued separately and bound separately.

As an analogy, DNS is the company phone directory. It tells you where to connect. A TLS certificate is an employee ID. Just because a letter arrives addressed to `example.com` does not mean the person receiving it has an ID that says `example.com`. The guard, meaning the TLS check, verifies the ID and rejects the connection if it is missing.

Also, because the handshake fails **before the HTTP request**, the backend application logs nothing. Debugging service code is pointless here. This is purely an infrastructure/certificate problem.

## Fix: Issue and bind the certificate

No matter where the server runs, the fix has the same shape.

### Certificate vs binding

These are separate things.

| Object | Meaning |
|------|------|
| **Certificate** | A signed file proving "this server is `example.com`" |
| **Binding** | Configuration saying "when requests arrive for this domain, present this certificate" |

Even if the certificate exists on the server, without a binding the server can still present the wrong default certificate.

### General fix process

1. **Issue a certificate that covers `example.com`.**
   - Let's Encrypt, usually with `certbot`
   - A hosting platform's managed certificate
   - A purchased certificate, such as `.pem` or `.pfx`

2. **Bind the certificate to the server/platform.**
   - Nginx: configure `ssl_certificate` and `ssl_certificate_key`
   - Apache: configure `SSLCertificateFile` and `SSLCertificateKeyFile`
   - Cloud platforms: choose and bind the certificate in the custom domain settings

### Nginx example

```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    ...
}
```

If using certbot:

```bash
certbot --nginx -d example.com
# certbot can update the nginx configuration automatically
```

### Azure example (App Service / Container Apps)

In the portal:

```
App resource -> Custom domains
  -> click "No binding" beside example.com
  -> Add binding
  -> Source = Create managed certificate
  -> TLS/SSL type = SNI SSL
  -> Add
```

**App Service:**

```bash
# 1. Create managed cert
az webapp config ssl create \
  --resource-group <resource-group> \
  --name <app-name> \
  --hostname example.com

# 2. Check thumbprint
az webapp config ssl list --resource-group <resource-group> \
  --query "[?subjectName=='example.com'].thumbprint" -o tsv

# 3. Bind it (SNI SSL)
az webapp config ssl bind \
  --resource-group <resource-group> \
  --name <app-name> \
  --certificate-thumbprint <thumbprint> \
  --ssl-type SNI
```

**Container Apps** (the certificate belongs to the Environment, not directly to the app):

```bash
az containerapp hostname add  -g <resource-group> -n <app-name> --hostname example.com
az containerapp hostname bind -g <resource-group> -n <app-name> --hostname example.com \
  --environment <environment> --validation-method CNAME
```

### Verify

The verification method is the same on any platform.

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -ext subjectAltName
```

If the output includes `example.com` in the SAN, the fix worked. `ERR_TLS_CERT_ALTNAME_INVALID` should no longer appear.

## Notes

1. **Bind the certificate where TLS actually terminates.** If a reverse proxy such as Nginx, Caddy, or a load balancer sits in front, the certificate belongs there. Binding it only on the backend app server will not help.

2. **DNS must be correct before certificate issuance can work.**
   - Root domain (`example.com`) -> A record
   - Subdomain (`www.example.com`) -> CNAME is possible
   - If a CAA record exists, it must allow the CA you use, for example `0 issue letsencrypt.org`

3. **Without a wildcard certificate, each hostname needs its own coverage.** `example.com` and `www.example.com` are separate names.

4. **Do not paper over this with `rejectUnauthorized: false`.** The error disappears, but TLS verification is disabled entirely. That exposes the webhook channel to impersonation attacks.

5. **Automatic renewal is conditional.** Whether you use Let's Encrypt or a managed certificate, renewal depends on the DNS record still existing and pointing at the server. If the A/CNAME record changes, renewal can fail.

---

## Three-line summary

1. `ERR_TLS_CERT_ALTNAME_INVALID` happens when the server certificate's SAN list does not contain the hostname the client connected to. It is platform-independent; routing a domain without binding a matching certificate is enough to trigger it.
2. DNS routing(domain -> server traffic) and TLS identity(proving the server owns that domain) are independent. Traffic arriving at a domain does not create a certificate.
3. The fix is to issue a certificate for the domain and bind it to the server. Since the failure happens before the HTTP request, debugging application code will not help.
