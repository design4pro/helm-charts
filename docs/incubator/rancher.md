# Rancher

``` bash
helm install rancher-stable/rancher --name rancher --namespace cattle-system --set hostname=rancher.design4.io --set ingress.tls.source=letsEncrypt --set letsEncrypt.email=rancher@design4.io
```
