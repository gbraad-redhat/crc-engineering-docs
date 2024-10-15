Add a self-signed certificate registry
======================================

CRC does not have any option to configure a self-signed registry.

> [!NOTE]
> For insecure registries (no valid TLS certificates, or HTTP-only), see [this page](./Adding-an-insecure-registry.md).


## Instructions
To provide the self-signed registry `my.self-signed.registry.com`:

> [!NOTE]
> The registry needs to be resolvable by DNS and reachable from the CRC VM.


1. Start the cluster and log in to it as `kubeadmin` via `oc`:
```bash
$ crc start
[...]
INFO You can now run 'crc console' and use these credentials to access the OpenShift web console 
Started the OpenShift cluster
WARN The cluster might report a degraded or error state. This is expected since several operators have been disabled to lower the resource usage. For more information, please consult the documentation

$ eval $(crc oc-env)
$ oc login -u kubeadmin -p <kubeadmin_password> https://api.crc.testing:6443
Login successful.

You have access to 51 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

1. Follow https://docs.openshift.com/container-platform/latest/openshift_images/image-configuration.html#images-configuration-file_image-configuration to make require changes in the cluster image resource.

2. SSH to the VM and update the registry cert file:

Ref: https://github.com/containers/image/blob/master/docs/containers-certs.d.5.md

```
<CRC-VM> $ sudo mkdir /etc/containers/certs.d/my.self-signed.registry.com

<CRC-VM> $ cat /etc/containers/certs.d/my.self-signed.registry.com/ca.crt
-----BEGIN CERTIFICATE-----
MIIC6jCCAdKgAwIBAgIBATANBgkqhkiG9w0BAQsFADAmMSQwIgYDVQQDDBtvcGVu
c2hpZnQtc2lnbmVyQDE1ODEyOTA4MTUwHhcNMjAwMjA5MjMyNjU0WhcNMjUwMjA3
MjMyNjU1WjAmMSQwIgYDVQQDDBtvcGVuc2hpZnQtc2lnbmVyQDE1ODEyOTA4MTUw
ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQC9GYTSeKJisiaUPVV8jlrm
Btx7cA6sFlPwjJB08G7eVdlBzjUwDZ59kTNlJbwzJXDcvIfYFabTt0G69diXOfmy
fQZvm5odVLARSVnDqBf3qIymiMMR8iWqxhwYkbB+Z5Dk8h2miR9FxyWg/q/pDw0G
yurtuOerMgZ0T5I0DSPnva1PXwJBV5lyR/65gC62F8H0K/MOInOfjdMifMoKdpBj
3o+tF1iv91mQztYC4y0G7Y3pq75bfJb1XQw0bqdYe4ULxDaZnAW7jRrvdiSSWbSd
zbGoZ2yFNIu+WKvUu8BOnUwFFVqLb8BLXtKbRxuQ
-----END CERTIFICATE-----


<CRC-VM> $ sudo systemctl restart crio
<CRC-VM> $ sudo systemctl restart kubelet
<CRC-VM> $ exit
```

3. If the self signed registry require authentication then you need to follow https://docs.openshift.com/container-platform/latest/openshift_images/managing-images/using-image-pull-secrets.html#images-allow-pods-to-reference-images-from-secure-registries_using-image-pull-secrets

4. Deploy app using the self signed registry.
```
$ oc new-app --docker-image=my.self-signed.registry.com/test-project1/httpd-example:latest --allow-missing-images --name=world
[...]

--> Creating resources ...
    deploymentconfig.apps.openshift.io "world" created
--> Success
    Run 'oc status' to view your app.

$ oc get pods
NAME             READY   STATUS      RESTARTS   AGE
world-1-6xbpb    1/1     Running     0          2m10s
world-1-deploy   0/1     Completed   0          2m19s
```