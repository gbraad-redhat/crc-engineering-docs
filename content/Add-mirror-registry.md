Add mirror registry
===================


1. Create `imageContentSourePolicy`
```bash
$ cat registryrepomirror.yaml 
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: ubi8repo
spec:
  repositoryDigestMirrors:
  - mirrors:
    - <my-repo-host>:<port>/ubi8-minimal
    source: registry.redhat.io/ubi8-minimal

$ oc apply -f registryrepomirror.yaml
```


2. Add registry to the OpenShift image config

  * https://docs.openshift.com/container-platform/latest/openshift_images/image-configuration.html#images-configuration-file_image-configuration
  * In case of insecure registry check https://github.com/code-ready/crc/wiki/Adding-an-insecure-registry
  * In case of self signed registry check https://github.com/code-ready/crc/wiki/Adding-a-self-signed-certificate-registry


3. SSH to the VM and perform following

> [!NOTE]
> For more info about registeries.conf check https://github.com/containers/image/blob/master/docs/containers-registries.conf.5.md
> or `man containers-registries.conf`

Here we are using the mirror registry which is self signed and behind the auth.
```bash
$ crc ip
192.168.64.92

$ ssh -i ~/.crc/machines/crc/id_rsa -o StrictHostKeyChecking=no core@192.168.64.92

<CRC-VM> $  cat /etc/containers/registries.conf 
unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]

[[registry]]
  prefix = ""
  location = "registry.redhat.io/ubi8-minimal"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "<your-mirror-registry>:<port>/ubi8-minimal"

```


4. If you need to have global pull secret then update the `/var/lib/kubelet/config.json` file inside the VM along with `pull-secret` on `openshift-config` namespace.
```bash
$ oc get secret pull-secret -n openshift-config --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode > pull-secret.json
$ oc registry login -a pull-secret.json --registry <your-mirror-registry>:<port> --auth-basic='<username>:<password>'
$ oc set data secret/pull-secret -n openshift-config --from-file=.dockerconfigjson=pull-secret.json
<CRC-VM> $ cat /var/lib/kubelet/config.json
```

> [!NOTE]
> This should have same content as `pull-secret.json` file


5. Restart the `kubelet and crio` service in the VM.
```bash
<CRC-VM> $ systemctl restart crio
<CRC-VM> $ systemctl restart kubelet
```

## References
  - https://docs.openshift.com/container-platform/latest/openshift_images/samples-operator-alt-registry.html#installation-creating-mirror-registry_samples-operator-alt-registry
  - https://docs.openshift.com/container-platform/latest/openshift_images/image-configuration.html#images-configuration-registry-mirror_image-configuration