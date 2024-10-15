Custom CA cert for proxy
========================


> [!NOTE]
> These steps are no longer needed, this is automated in newer CRC releases. This page is only useful for historical documentation

---

- Start the CRC with proxy setting as mentioned [here](https://crc-org.github.io/crc/#starting-codeready-containers-behind-proxy_gsg).
- Create a `user-ca-bundle.yaml` resource as instructed by the [OpenShift docs](https://docs.openshift.com/container-platform/latest/networking/enable-cluster-wide-proxy.html#nw-proxy-configure-object_config-cluster-wide-proxy):

```bash
$ cat user-ca-bundle.yaml 
apiVersion: v1
data:
ca-bundle.crt: |
-----BEGIN CERTIFICATE-----
.
.
.
-----END CERTIFICATE-----
kind: ConfigMap
metadata:
name: user-ca-bundle
namespace: openshift-config
```

- Apply the resource to cluster:
```bash
$ oc apply user-ca-bundle.yaml
```

- Check the status of operators (most of then will go to progressing state and the come back as available:
```bash
$ oc get co
```

- SSH to crc VM and add the custom cert and run update-ca-trust:
```bash
$ crc ip
$ ssh -i ~/.crc/machines/crc/id_rsa -o StrictHostKeyChecking=no -o UserKnownHostsFile=/dev/null core@<crc_ip>
$ sudo vi /etc/pki/ca-trust/source/anchors/openshift-config-user-ca-bundle.crt
$ sudo update-ca-trust
$ sudo systemctl restart crio
$ sudo systemctl restart kubelet
```

- Exit from the crc vm and check the operators:
```bash
$ oc get co
```