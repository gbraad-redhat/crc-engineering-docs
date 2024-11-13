Change the domain for CRC
=========================


We have default route for apps is `apps-crc.testing` and for API server `api.crc.testing`, Some users want to use a different domain and as long as it resolve the Instance IP, a user should able to change the domain name.

Changes to the `ingress` domain are not permitted as a day-2 operation https://access.redhat.com/solutions/4853401 

What we have to do is add component routes and `appDomain` to ingress resource to make our custom domain to work with cluster.
  - https://docs.openshift.com/container-platform/latest/rest_api/config_apis/ingress-config-openshift-io-v1.html#spec-componentroutes
  - https://docs.openshift.com/container-platform/latest/web_console/customizing-the-web-console.html#customizing-the-console-route_customizing-web-console
  - https://docs.openshift.com/container-platform/latest/authentication/configuring-internal-oauth.html#customizing-the-oauth-server-url_configuring-internal-oauth
  - https://docs.openshift.com/container-platform/latest/security/certificates/api-server.html#customize-certificates-api-add-named_api-server-certificates

In these steps we are using `<VM_IP>.nip.io` on Linux box where `ip` is set to `192.168.130.11` in case of user mode networking, you can check it with `crc ip` command. 

> [!NOTE]
> Whatever domain you want to use make sure it is resolvable inside cluster. Otherwise after all those steps you will see following warning for oauth and console operator because `console-openshift-console.apps.192.168.130.11.nip.io` not able to be resolved inside cluster.

    RouteHealthAvailable: failed to GET route (https://console-openshift-console.apps.192.168.130.11.nip.io): Get "https://console-openshift-console.apps.192.168.130.11.nip.io": dial tcp: lookup console-openshift-console.apps.192.168.130.11.nip.io on 10.217.4.10:53: server misbehaving


## Instructions

1. Create a custom cert/key pair for the domain
```bash
$ openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout nip.key -out nip.crt -subj "/CN=192.168.130.11.nip.io" -addext "subjectAltName=DNS:apps.192.168.130.11.nip.io,DNS:*.apps.192.168.130.11.nip.io,DNS:api.192.168.130.11.nip.io"
```

1. Create TLS secret using those cert/key pair (here named `nip-secret`)
```bash
$ oc create secret tls nip-secret --cert=nip.crt --key=nip.key -n openshift-config
```

1. Create an ingress patch which have details about component routes and `appsDomain` and apply it.
```bash
$ cat <<EOF > ingress-patch.yaml
spec:
  appsDomain: apps.192.168.130.11.nip.io
  componentRoutes:
  - hostname: console-openshift-console.apps.192.168.130.11.nip.io
    name: console
    namespace: openshift-console
    servingCertKeyPairSecret:
      name: nip-secret
  - hostname: oauth-openshift.apps.192.168.130.11.nip.io
    name: oauth-openshift
    namespace: openshift-authentication
    servingCertKeyPairSecret:
      name: nip-secret
EOF

$ oc patch ingresses.config.openshift.io cluster --type=merge --patch-file=ingress-patch.yaml
```

4. Create a patch request for apiserver to add our custom certificate as serving cert.
```bash
$ oc patch apiserver cluster --type=merge -p '{"spec":{"servingCerts": {"namedCertificates":[{"names":["api.192.168.130.11.nip.io"],"servingCertificate": {"name": "nip-secret"}}]}}}'
```

5. Update the old routes host to new one.
```bash
$ oc patch -p '{"spec": {"host": "default-route-openshift-image-registry.192.168.130.11.nip.io"}}' route default-route -n openshift-image-registry --type=merge
```

6. Keep looking at `oc get co` to make sure everything is available.
```bash
# Wait till all the operator reconcile and in Available state (no progressing or degraded state)
$ oc get co
NAME                                       VERSION   AVAILABLE   PROGRESSING   DEGRADED   SINCE   MESSAGE
authentication                             4.11.3    True        False         False      73m     
config-operator                            4.11.3    True        False         False      5d19h   
console                                    4.11.3    True        False         False      73m     
dns                                        4.11.3    True        False         False      92m     
etcd                                       4.11.3    True        False         False      5d19h   
image-registry                             4.11.3    True        False         False      87m     
ingress                                    4.11.3    True        False         False      5d19h   
kube-apiserver                             4.11.3    True        False         False      5d19h   
kube-controller-manager                    4.11.3    True        False         False      5d19h   
kube-scheduler                             4.11.3    True        False         False      5d19h   
machine-api                                4.11.3    True        False         False      5d19h   
machine-approver                           4.11.3    True        False         False      5d19h   
machine-config                             4.11.3    True        False         False      5d19h   
marketplace                                4.11.3    True        False         False      5d19h   
network                                    4.11.3    True        False         False      5d19h   
node-tuning                                4.11.3    True        False         False      5d19h   
openshift-apiserver                        4.11.3    True        False         False      80m     
openshift-controller-manager               4.11.3    True        False         False      87m     
openshift-samples                          4.11.3    True        False         False      5d19h   
operator-lifecycle-manager                 4.11.3    True        False         False      5d19h   
operator-lifecycle-manager-catalog         4.11.3    True        False         False      5d19h   
operator-lifecycle-manager-packageserver   4.11.3    True        False         False      92m     
service-ca                                 4.11.3    True        False         False      5d19h   
```

Try to login to cluster using the new api URI
```bash
# Get the kubeadmin user password
$ crc console --credentials

$ oc login -u kubeadmin -p <password> https://api.192.168.130.11.nip.io:6443
The server is using a certificate that does not match its hostname: x509: certificate is valid for kubernetes, kubernetes.default, kubernetes.default.svc, kubernetes.default.svc.cluster.local, openshift, openshift.default, openshift.default.svc, openshift.default.svc.cluster.local, 172.25.0.1, not api.192.168.130.11.nip.io
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y
Login successful.

You have access to 57 projects, the list has been suppressed. You can list all projects with 'oc projects'

Using project "default".
```

Try to create a sample app and expose the route
```bash
$ oc new-project demo
$ oc new-app ruby~https://github.com/sclorg/ruby-ex.git
$ oc expose svc/ruby-ex
$ oc get route
NAME      HOST/PORT                            PATH   SERVICES   PORT       TERMINATION   WILDCARD
ruby-ex   ruby-ex-demo.192.168.130.11.nip.io          ruby-ex    8080-tcp                 None

$ curl -Ik ruby-ex-demo.192.168.130.11.nip.io
HTTP/1.1 200 OK
content-type: text/html
content-length: 39559
set-cookie: 5735a0b0e41f7362ba688320968404a3=4268ca9aa18f871004be9c1bd0112787; path=/; HttpOnly
cache-control: private

```