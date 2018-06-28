## Troubleshooting tips for Cert-Manager

Most of the common issues seem to come from slow DNS resolution. If you are configuring an A record for your domain around the same time as deployment then you will probably run into issues when letsencrypt attempts to verify the domain. If the domain is not resolving yet, then we can assume that the challenge file is not reachable.

After deploying in Kubernetes we will want to take a look at ingress resources. The ingress resources will be used by the ingress controller to configure nginx, and then reload the configuration. In Bluemix we end up using the automatically provisioned LoadBalancer Service which is in the kube-system namespace.

First we need to see which ingress resources were created. We can do so with the command below. If we are checking in a different namespace then we need to append -n NNAMESPACE_NAME

```bash
kubectl get ingress
```

Find the name of the resource that was recently created and then describe it.

```bash
kubectl describe ingress INGRESS_NAME
```

Check for valuable information in Events. Normally we'll see something like "failed to apply ingress resource" in the message field, and if we check the "Reason" field we'll actually get a useful error message. This is great coming from OpenStack where an error message felt more like a vague comment about something not working.

```bash
Events:
  Type     Reason             Age   From                                                             Message
  ----     ------             ----  ----                                                             -------
  Warning  TLSSecretNotFound  3s    public-cr0ba8157fd1a6454ca7ba3125b9b44ff6-alb1-5895555f68-bl976  Failed to apply ingress resource.
  Warning  TLSSecretNotFound  3s    public-cr0ba8157fd1a6454ca7ba3125b9b44ff6-alb1-5895555f68-25nhq  Failed to apply ingress resource.
```

After figuring out that the TLS Secret may be missing we'll need to see what the expected resource is named. We name the secret in our ingress resource, so we'll check there first.

```bash
kubectl get ingress INGRESS_NAME -o yaml
```

We use the output option to specify yaml so we can read the configuration file. You can also use describe instead of using "get" with "-o yaml", we'll just see the output in a different format. In our case our secret name is lp-mpetason-com-tls1.

```yaml
  tls:
  - hosts:
    - lp.mpetason.com
    secretName: lp-mpetason-com-tls1
```

Check for configured secrets to see if the secret is configured, or if it has a different name for some reason. If we are having trouble with our deployment then it may not have been created. In order to get the file created we would need for the Issuer and the Cert to finish out getting configured.

```bash
kubectl get issuer
kubectl describe issuer ISSUER_NAME
```

We should be able to find error message in Events. Most of the error messages about the Issuer have been related to the acme endpoint. There may be other issues that can come up, however I haven't seen them enough to help troubleshoot - yet. For the most part you can try to resolve the issues you see in the Event info or Status.

If our issuer is working without issues we should see something like:

```bash
Status:
  Acme:
    Uri:  https://acme-v01.api.letsencrypt.org/acme/reg/<NUMBERS>
  Conditions:
    Last Transition Time:  2018-06-14T18:12:24Z
    Message:               The ACME account was registered with the ACME server
    Reason:                ACMEAccountRegistered
    Status:                True
    Type:                  Ready
Events:                    <none>
```

As of this post we should probably be using acme-v02 instead. If you run into errors about the version, go ahead and change it.

Next up we need to take a look at the cert and see what the status is.

```bash
kubectl get cert
kubectl describe cert CERT_NAME
```

Here we can run into a few other issues - such as rate limiting if we've tried to register a ton in a short period.

Normally if the issuer is working, and DNS is resolving, we should be able to get a cert. After we confirm that we have a cert via the Describe on the cert, we'll need to take a look at secrets to verify it was created.

```bash
kubectl get secret
```

If the secret exists we can go back over to the ingress resource to see if the ingress controller was able to load our cert.

```bash
  Warning  TLSSecretNotFound  26m   public-cr0ba8157fd1a6454ca7ba3125b9b44ff6-alb1-5895555f68-bl976  Failed to apply ingress resource.
  Warning  TLSSecretNotFound  26m   public-cr0ba8157fd1a6454ca7ba3125b9b44ff6-alb1-5895555f68-25nhq  Failed to apply ingress resource.
  Normal   Success            11s   public-cr0ba8157fd1a6454ca7ba3125b9b44ff6-alb1-5895555f68-25nhq  Successfully applied ingress resource.
  Normal   Success            11s   public-cr0ba8157fd1a6454ca7ba3125b9b44ff6-alb1-5895555f68-bl976  Successfully applied ingress resource.
```

Success! Now we can hit the site and see if the cert worked properly.