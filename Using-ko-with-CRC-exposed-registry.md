Using `ko` with CRC exposed registry
====================================


By default CRC expose the internal registry `default-route-openshift-image-registry.apps-crc.testing` for use. But this registry route uses a self signed certificate. To use it with `ko` you need to follow some manual steps.


## Instructions

1. Download the route ca cert which used to sign the registry route.

```bash
$ oc extract secret/router-ca --keys=tls.crt -n openshift-ingress-operator
```

2. Use this cert to login to registry using docker.
```bash
$ sudo mkdir -p /etc/docker/certs.d/default-route-openshift-image-registry.apps-crc.testing
$ sudo cp tls.crt /etc/docker/certs.d/default-route-openshift-image-registry.apps-crc.testing
$ docker login -u kubeadmin -p $(oc whoami -t)  default-route-openshift-image-registry.apps-crc.testing
```

3. `ko` doesn't have a way to specify the registry cert https://github.com/google/ko/issues/142 so for **Linux** you can use `SSL_CERT_FILE` environment variable to specify it and for **MacOS** you need to add it to the keyring https://github.com/google/go-containerregistry/issues/211 

```bash
$ export SSL_CERT_FILE=/etc/docker/certs.d/default-route-openshift-image-registry.apps-crc.testing/tls.crt
```

4. Now you can use `ko` with internal registry to push the image (Using tekton example here).
```bash
$ git clone https://github.com/redhat-developer/tekton-hub.git
$ cd /tekton-hub/backend/api
$ KO_DOCKER_REPO=default-route-openshift-image-registry.apps-crc.testing/tekton-hub ko apply -f config/
$ KO_DOCKER_REPO=default-route-openshift-image-registry.apps-crc.testing/tekton-hub ko apply -f config/ 
2020/03/16 12:15:55 Using base gcr.io/distroless/static:latest for github.com/redhat-developer/tekton-hub/backend/api/cmd/api
namespace/tekton-hub unchanged
secret/db configured
persistentvolumeclaim/db unchanged
deployment.apps/db unchanged
service/db unchanged
secret/api configured
2020/03/16 12:15:58 Building github.com/redhat-developer/tekton-hub/backend/api/cmd/api
2020/03/16 12:16:05 Publishing default-route-openshift-image-registry.apps-crc.testing/tekton-hub/api-b786b59ff17bae65aa137e516553ea05:latest
2020/03/16 12:16:05 Published default-route-openshift-image-registry.apps-crc.testing/tekton-hub/api-b786b59ff17bae65aa137e516553ea05@sha256:34f4ad707c69fc7592ae3f92f62cf5741468fc7083d0662dd67dc15b08cf5128
deployment.apps/api unchanged
route.route.openshift.io/api unchanged
service/api unchanged
```

5. By default the exposed registry is behind the auth so you will see following.
```bash
$ oc get all -n tekton-hub
NAME                       READY   STATUS             RESTARTS   AGE
pod/api-6cf586db66-4djtr   0/1     ImagePullBackOff   0          88m
pod/db-7f6bdf76c8-g6g84    1/1     Running            2          3d

NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service/api   NodePort    172.30.62.51    <none>        5000:32601/TCP   3d
service/db    ClusterIP   172.30.16.148   <none>        5432/TCP         3d

NAME                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/api   0/1     1            0           88m
deployment.apps/db    1/1     1            1           3d

NAME                             DESIRED   CURRENT   READY   AGE
replicaset.apps/api-6cf586db66   1         1         0       88m
replicaset.apps/db-7f6bdf76c8    1         1         1       3d

NAME                                                                  IMAGE REPOSITORY                                                                                          TAGS     UPDATED
imagestream.image.openshift.io/api-b786b59ff17bae65aa137e516553ea05   default-route-openshift-image-registry.apps-crc.testing/tekton-hub/api-b786b59ff17bae65aa137e516553ea05   latest   2 hours ago

NAME                           HOST/PORT                         PATH   SERVICES   PORT    TERMINATION     WILDCARD
route.route.openshift.io/api   api-tekton-hub.apps-crc.testing          api        <all>   edge/Redirect   None

$ oc get events -n tekton-hub
LAST SEEN   TYPE      REASON              OBJECT                      MESSAGE
<unknown>   Normal    Scheduled           pod/api-6cf586db66-4djtr    Successfully assigned tekton-hub/api-6cf586db66-4djtr to crc-jccc5-master-0
87m         Normal    Pulling             pod/api-6cf586db66-4djtr    Pulling image "default-route-openshift-image-registry.apps-crc.testing/tekton-hub/api-b786b59ff17bae65aa137e516553ea05@sha256:34f4ad707c69fc7592ae3f92f62cf5741468fc7083d0662dd67dc15b08cf5128"
87m         Warning   Failed              pod/api-6cf586db66-4djtr    Failed to pull image "default-route-openshift-image-registry.apps-crc.testing/tekton-hub/api-b786b59ff17bae65aa137e516553ea05@sha256:34f4ad707c69fc7592ae3f92f62cf5741468fc7083d0662dd67dc15b08cf5128": rpc error: code = Unknown desc = Error reading manifest sha256:34f4ad707c69fc7592ae3f92f62cf5741468fc7083d0662dd67dc15b08cf5128 in default-route-openshift-image-registry.apps-crc.testing/tekton-hub/api-b786b59ff17bae65aa137e516553ea05: unauthorized: authentication required
```

6. You need to add the docker registry secret to the `tekton-hub` namespace.
```bash
$ oc create secret docker-registry internal-registry --docker-server=default-route-openshift-image-registry.apps-crc.testing --docker-username=kubeadmin --docker-password=$(oc whoami -t) --docker-email=abc@gmail.com -n  tekton-hub
$ oc secrets link default internal-registry --for=pull -n  tekton-hub
$ oc secrets link builder internal-registry -n  tekton-hub
$ KO_DOCKER_REPO=default-route-openshift-image-registry.apps-crc.testing/tekton-hub ko apply -f config/ 
<== Remove old ImagePullBackOff pod ==>
$ oc delete pod/api-6cf586db66-4djtr  -n  tekton-hub
```