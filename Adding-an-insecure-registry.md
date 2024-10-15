Adding an insecure registry
===========================

CRC does not have a configuration option to provide an insecure registry. An insecure registry is a registry without a valid TLS certificate, or one which only supports HTTP connections.

> [!NOTE]
> For self-signed registries, see [this page](./Adding-a-self-signed-certificate-registry.md).


## Instructions
To provide the insecure registry `my.insecure.registry.com:8888`:

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

2. Patch the `image.config.openshift.io` resource and add your insecure registry details:
```bash
$ oc patch --type=merge --patch='{
  "spec": {
    "registrySources": {
      "insecureRegistries": [
      "my.insecure.registry.com:8888"
      ]
    }
  }
}' image.config.openshift.io/cluster
image.config.openshift.io/cluster patched
```

3. SSH to the VM and update the `/etc/containers/registries.conf` file to add details about the insecure registry:
```bash
$ crc ip
192.168.64.92

$ ssh -i ~/.crc/machines/crc/id_rsa -o StrictHostKeyChecking=no core@192.168.64.92

<CRC-VM> $ sudo cat /etc/containers/registries.conf 
unqualified-search-registries = ['registry.access.redhat.com', 'docker.io']

[[registry]]
  location = "my.insecure.registry.com:8888"
  insecure = true
  blocked = false
  mirror-by-digest-only = false
  prefix = ""

<CRC-VM> $ sudo systemctl restart crio
<CRC-VM> $ sudo systemctl restart kubelet
<CRC-VM> $ exit
```

4. Deploy your workload using the insecure registry:
```bash
$ cat test.yaml 
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: test
  name: test
spec:
  containers:
  - image: my.insecure.registry.com:8888/test/testimage
    name: test
    command: ['sh', '-c', 'echo Hello Kubernetes! && sleep 3600']

$ oc create -f test.yaml
pod/test created

$ watch oc get events
46s         Normal    Scheduled                 pod/test                        Successfully assigned default/test to crc-shdl4-master-0
38s         Normal    Pulling                   pod/test                        Pulling image "my.insecure.registry.com:8888/test/testimage"
15s         Normal    Pulled                    pod/test                        Successfully pulled image "my.insecure.registry.com:8888/test/testimage"
15s         Normal    Created                   pod/test                        Created container test
15s         Normal    Started                   pod/test                        Started container test

$ oc get pods
NAME   READY   STATUS    RESTARTS   AGE
test   1/1     Running   0          8m40s
```

To perform each steps as bash script, you can try something following.
```bash
$ oc login -u kubeadmin -p $(crc console --credentials | awk -F "kubeadmin" '{print $2}' | cut -c 5- | rev | cut -c31- | rev) https://api.crc.testing:6443

$ oc patch --type=merge --patch='{
  "spec": {
    "registrySources": {
      "insecureRegistries": [
      "YOUR_REGISTRY"
      ]
    }
  }
}' image.config.openshift.io/cluster

$ ssh -i ~/.crc/machines/crc/id_rsa -o StrictHostKeyChecking=no core@$(crc ip) << EOF
  sudo echo " " | sudo tee -a /etc/containers/registries.conf
  sudo echo "[[registry]]" | sudo tee -a /etc/containers/registries.conf
  sudo echo "  location = \"YOUR_REGISTRY\"" | sudo tee -a /etc/containers/registries.conf
  sudo echo "  insecure = true" | sudo tee -a /etc/containers/registries.conf
  sudo echo "  blocked = false" | sudo tee -a /etc/containers/registries.conf
  sudo echo "  mirror-by-digest-only = false" | sudo tee -a /etc/containers/registries.conf
  sudo echo "  prefix = \"\"" | sudo tee -a /etc/containers/registries.conf
  sudo systemctl restart crio
  sudo systemctl restart kubelet
EOF
```