Add another user to the cluster
===============================

For CRC we use htpasswd method to manage the users in the OpenShift cluster https://docs.openshift.com/container-platform/latest/authentication/identity_providers/configuring-htpasswd-identity-provider.html#add-identity-provider_configuring-htpasswd-identity-provider, by default we have `developer` and `kubeadmin` user which is created at disk creation time and `kubeadmin` user has the `cluster-admin` role.

If you want to add a new user to cluster following steps should work.

> [!NOTE]
> Make sure you have the `htpasswd` command.
> In Fedora it is provided by `httpd-tools` package

```bash
$ export HTPASSWD_FILE=/tmp/htpasswd

$ htpasswd -c -B -b $HTPASSWD_FILE user1 password1
$ htpasswd -b $HTPASSWD_FILE user2 password2

$ cat $HTPASSWD_FILE
user1:$2y$05$4QxnejXAJ2nmnVFXlNXn/ega9BUrKbaGLpOtdS2LJXmbOECXWSVDa
user2:$apr1$O9jL/dfz$qXs216/W8Waw2.p7rvhJR.
```

> [!WARNING]
> Make sure the existing `developer` and `kubeadmin` users are part of `htpasswd` file because `kubeadmin` has the cluster admin role.

```
$ oc get secrets htpass-secret -n openshift-config -ojsonpath='{.data.htpasswd}' | base64 -d >> htpasswd 

$ oc create secret generic htpass-secret --from-file=$HTPASSWD_FILE -n openshift-config --dry-run -o yaml > /tmp/htpass-secret.yaml
$ oc replace -f /tmp/htpass-secret.yaml
```


Check the auth pods which are going to recreated because of this config change.
```bash
$ oc get pods -n openshift-authentication
$ oc get pods -n openshift-authentication
NAME                               READY   STATUS    RESTARTS   AGE
oauth-openshift-7f4994c969-8fz44   1/1     Running   0          11s
oauth-openshift-7f4994c969-mjrjc   1/1     Running   0          11s
```