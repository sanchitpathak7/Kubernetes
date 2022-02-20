##Audit Logging using Kubeadm
Single Node Cluster

```
controlplane $ kubectl get nodes
NAME           STATUS   ROLES                  AGE   VERSION
controlplane   Ready    control-plane,master   16d   v1.23.1
```

```
controlplane $ mkdir /etc/kubernetes/audit-logs
controlplane $ ls -lrt /etc/kubernetes/audit-logs
total 0
```

```
controlplane $ cat /etc/kubernetes/audit-policy/policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:

# log Secret & ConfigMap resources audits, level Metadata
- level: Metadata
  resources:
  - group: "" # core API group
    resources: ["configmaps", "secrets"]

# log node related audits, level RequestResponse
- level: RequestResponse
  userGroups: ["system:nodes"]

# log namespace resource audits, level RequestResponse
- level: RequestResponse
  resources:
  - group: ""
    resources: ["namespaces"]

# log pods changes in the namespace kube-system, level Request
- level: Request
  resources:
  - group: ""
    resources: ["pods"]
  namespace: ["kube-system"]


# A catch-all rule to log all other requests at the Metadata level.
- level: Metadata
  omitStages: 
    - "RequestReceived"
```


```
controlplane $ cat /etc/kubernetes/manifests/kube-apiserver.yaml 
apiVersion: v1
kind: Pod
metadata:
  annotations:
    kubeadm.kubernetes.io/kube-apiserver.advertise-address.endpoint: 172.30.1.2:6443
  creationTimestamp: null
  labels:
    component: kube-apiserver
    tier: control-plane
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
...
    - --audit-policy-file=/etc/kubernetes/audit-policy/policy.yaml
    - --audit-log-path=/etc/kubernetes/audit-logs/audit.log
    - --audit-log-maxsize=7
    - --audit-log-maxbackup=2
...
    volumeMounts:
...
    - mountPath: /etc/kubernetes/audit-policy/policy.yaml
      name: audit
      readOnly: true
    - mountPath: /etc/kubernetes/audit-logs/
      name: audit-log
      readOnly: false
...
  volumes:
  - hostPath:
      path: /etc/kubernetes/audit-policy/policy.yaml
      type: File
    name: audit
  - hostPath:
      path: /etc/kubernetes/audit-logs/
      type: DirectoryOrCreate
    name: audit-log
 ```

```
 controlplane $ crictl ps | grep kube-apiserver
a7c151dff1d2e       b6d7abedde399       About a minute ago   Running             kube-apiserver            7                   cade8bb19f703
```

```
controlplane $ ls -lrt /etc/kubernetes/audit-logs
total 2080
-rw------- 1 root root 2127740 Feb 20 03:38 audit.log
```

```
controlplane $ kubectl create namespace testaudit
namespace/testaudit created
```

```
controlplane $ cat /etc/kubernetes/audit-logs/audit.log
{"kind":"Event","apiVersion":"audit.k8s.io/v1","level":"RequestResponse","auditID":"76c364a9-3040-49df-a53e-7489452f30a1","stage":"ResponseComplete","requestURI":"/api/v1/namespaces?fieldManager=kubectl-create","verb":"create","user":{"username":"kubernetes-admin","groups":["system:masters","system:authenticated"]},"sourceIPs":["172.30.1.2"],"userAgent":"kubectl/v1.23.1 (linux/amd64) kubernetes/86ec240","objectRef":{"resource":"namespaces","name":"testaudit","apiVersion":"v1"},"responseStatus":{"metadata":{},"code":201},"requestObject":{"kind":"Namespace","apiVersion":"v1","metadata":{"name":"testaudit","creationTimestamp":null,"labels":{"kubernetes.io/metadata.name":"testaudit"}},"spec":{},"status":{"phase":"Active"}},"responseObject":{"kind":"Namespace","apiVersion":"v1","metadata":{"name":"testaudit","uid":"b3bd492b-debc-40d4-bff8-2da0dbb04be4","resourceVersion":"5409","creationTimestamp":"2022-02-20T03:39:41Z","labels":{"kubernetes.io/metadata.name":"testaudit"},"managedFields":[{"manager":"kubectl-create","operation":"Update","apiVersion":"v1","time":"2022-02-20T03:39:41Z","fieldsType":"FieldsV1","fieldsV1":{"f:metadata":{"f:labels":{".":{},"f:kubernetes.io/metadata.name":{}}}}}]},"spec":{"finalizers":["kubernetes"]},"status":{"phase":"Active"}},"requestReceivedTimestamp":"2022-02-20T03:39:41.238907Z","stageTimestamp":"2022-02-20T03:39:41.249782Z","annotations":{"authorization.k8s.io/decision":"allow","authorization.k8s.io/reason":""}}
```
