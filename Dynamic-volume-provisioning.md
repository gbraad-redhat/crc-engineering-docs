Dynamic volume provisioning
===========================

By default in crc we don't support dynamic volume provision because of the hostPath volumes. https://github.com/rancher/local-path-provisioner have a way to use hostPath for create local provisioner to use as dynamic one. This make this work for OpenShift some small changes are required.


## Instructions for local-path-provisioner
Start crc as usual way and wait till cluster is up.

> [!NOTE]
> This uses https://github.com/rancher/local-path-provisioner

```bash
$ oc login -u kubeadmin -p <passwd> https://api.crc.testing:6443
$ oc new-project local-path-storage
$ oc create serviceaccount local-path-provisioner-service-account -n local-path-storage
$ oc adm policy add-scc-to-user hostaccess -z local-path-provisioner-service-account -n local-path-storage
$ cat <<EOF | oc apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: local-path-provisioner-role
rules:
- apiGroups: [""]
  resources: ["nodes", "persistentvolumeclaims"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["endpoints", "persistentvolumes", "pods"]
  verbs: ["*"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["create", "patch"]
- apiGroups: ["storage.k8s.io"]
  resources: ["storageclasses"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: local-path-provisioner-bind
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: local-path-provisioner-role
subjects:
- kind: ServiceAccount
  name: local-path-provisioner-service-account
  namespace: local-path-storage
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: local-path-provisioner
  namespace: local-path-storage
spec:
  replicas: 1
  selector:
    matchLabels:
      app: local-path-provisioner
  template:
    metadata:
      labels:
        app: local-path-provisioner
    spec:
      serviceAccountName: local-path-provisioner-service-account
      containers:
      - name: local-path-provisioner
        image: rancher/local-path-provisioner:v0.0.12
        imagePullPolicy: IfNotPresent
        command:
        - local-path-provisioner
        - --debug
        - start
        - --config
        - /etc/config/config.json
        volumeMounts:
        - name: config-volume
          mountPath: /etc/config/
        env:
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
      volumes:
        - name: config-volume
          configMap:
            name: local-path-config
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-path
provisioner: rancher.io/local-path
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: local-path-config
  namespace: local-path-storage
data:
  config.json: |-
        {
                "nodePathMap":[
                {
                        "node":"DEFAULT_PATH_FOR_NON_LISTED_NODES",
                        "paths":["/mnt/pv-data"]
                }
                ]
        }
EOF
```

Check that the provisioner pod is running
```bash
 $ oc get all -n local-path-storage 
NAME                                          READY   STATUS    RESTARTS   AGE
pod/local-path-provisioner-58b55cb6b6-rn2vd   1/1     Running   0          15m

NAME                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/local-path-provisioner   1/1     1            1           15m

NAME                                                DESIRED   CURRENT   READY   AGE
replicaset.apps/local-path-provisioner-58b55cb6b6   1         1         1       15m
```

## Instructions for hostpath-provisioner

> [!NOTE]
> This uses https://github.com/kubevirt/hostpath-provisioner/

```bash
oc apply -f 'https://raw.githubusercontent.com/kubevirt/hostpath-provisioner/main/deploy/kubevirt-hostpath-security-constraints.yaml'

cat <<EOF | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: kubevirt-hostpath-provisioner
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: kubevirt-hostpath-provisioner
provisioner: kubevirt.io/hostpath-provisioner
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubevirt-hostpath-provisioner
subjects:
- kind: ServiceAccount
  name: kubevirt-hostpath-provisioner-admin
  namespace: kubevirt-hostpath-provisioner
roleRef:
  kind: ClusterRole
  name: kubevirt-hostpath-provisioner
  apiGroup: rbac.authorization.k8s.io
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: kubevirt-hostpath-provisioner
rules:
  - apiGroups: [""]
    resources: ["nodes"]
    verbs: ["get"]
  - apiGroups: [""]
    resources: ["persistentvolumes"]
    verbs: ["get", "list", "watch", "create", "delete"]
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "update"]

  - apiGroups: ["storage.k8s.io"]
    resources: ["storageclasses"]
    verbs: ["get", "list", "watch"]

  - apiGroups: [""]
    resources: ["events"]
    verbs: ["list", "watch", "create", "update", "patch"]
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kubevirt-hostpath-provisioner-admin
  namespace: kubevirt-hostpath-provisioner
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kubevirt-hostpath-provisioner
  labels:
    k8s-app: kubevirt-hostpath-provisioner
  namespace: kubevirt-hostpath-provisioner
spec:
  selector:
    matchLabels:
      k8s-app: kubevirt-hostpath-provisioner
  template:
    metadata:
      labels:
        k8s-app: kubevirt-hostpath-provisioner
    spec:
      serviceAccountName: kubevirt-hostpath-provisioner-admin
      containers:
        - name: kubevirt-hostpath-provisioner
          image: quay.io/kubevirt/hostpath-provisioner
          imagePullPolicy: Always
          env:
            - name: USE_NAMING_PREFIX
              value: "false" # change to true, to have the name of the pvc be part of the directory
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: PV_DIR
              value: /var/hpvolumes
          volumeMounts:
            - name: pv-volume # root dir where your bind mounts will be on the node
              mountPath: /var/hpvolumes
              #nodeSelector:
              #- name: xxxxxx
      volumes:
        - name: pv-volume
          hostPath:
            path: /mnt/pv-data
EOF
```

Check that the provisioner pod is running
```
$ oc get all -n kubevirt-hostpath-provisioner
NAME                                      READY   STATUS    RESTARTS   AGE
pod/kubevirt-hostpath-provisioner-xw777   1/1     Running   0          41m

NAME                                           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/kubevirt-hostpath-provisioner   1         1         1       1            1           <none>          41m
```