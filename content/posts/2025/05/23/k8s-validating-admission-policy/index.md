---
title: "K8s Validating Admission Policyを試す"
date: "2025-05-23T23:18:34+09:00"
draft: false
---

Kubernetes でマニフェストを適用する際に、独自のバリデーションをかけることができる Validating Admission Policy を試してみます。

https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/

## ClusterIP の Service のみ許可する Policy を作成する

「Service は ClusterIP のみ定義できる」というバリデーションを設定してみます。

こんな感じで services リソースで spec.type が ClusterIP なら Fail とします。

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicy
metadata:
  name: "only-cluster-ip-policy"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
      - apiGroups: [""]
        apiVersions: ["v1"]
        operations: ["CREATE", "UPDATE"]
        resources: ["services"]
  validations:
    - expression: "object.spec.type == 'ClusterIP'"
EOF
)
```

ポリシーを Binding します。

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "only-cluster-ip-policy-binding"
spec:
  policyName: "only-cluster-ip-policy"
  validationActions: [Deny]
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: test
EOF
)
```

ポリシーが適用されるネームスペースを作成します。

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: v1
kind: Namespace
metadata:
  name: only-cluster-ip-policy-ns
  labels:
    environment: test
EOF
)
```

## ClusterIP な Service を作成してみる

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: v1
kind: Service
metadata:
  name: cluster-ip-svc
  namespace: only-cluster-ip-policy-ns
spec:
  type: ClusterIP
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
)
```

作成できました。

```bash
hum@ryzen5pc:~$ kubectl get svc -n only-cluster-ip-policy-ns
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
cluster-ip-svc   ClusterIP   10.106.82.125   <none>        80/TCP    8s
```

## NodePort な Service を作成してみる

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: v1
kind: Service
metadata:
  name: node-port-svc
  namespace: only-cluster-ip-policy-ns
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
)
```

以下のようなエラーが出て作成できませんでした。

```
The services "node-port-svc" is invalid: : ValidatingAdmissionPolicy 'only-cluster-ip-policy' with binding 'only-cluster-ip-policy-binding' denied request: failed expression: object.spec.type == 'ClusterIP'
```

```bash
hum@ryzen5pc:~$ kubectl get svc -n only-cluster-ip-policy-ns
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
cluster-ip-svc   ClusterIP   10.106.82.125   <none>        80/TCP    86s
```

## LoadBalancer な Service を作成してみる

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: v1
kind: Service
metadata:
  name: load-balancer-svc
  namespace: only-cluster-ip-policy-ns
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
)
```

こちらも NodePort と同じくエラーになり作成できませんでした。

```
The services "load-balancer-svc" is invalid: : ValidatingAdmissionPolicy 'only-cluster-ip-policy' with binding 'only-cluster-ip-policy-binding' denied request: failed expression: object.spec.type == 'ClusterIP'
```

```bash
hum@ryzen5pc:~$ kubectl get svc -n only-cluster-ip-policy-ns
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
cluster-ip-svc   ClusterIP   10.106.82.125   <none>        80/TCP    3m16s
```

## ClusterIP から NodePort に変更してみる

```bash
hum@ryzen5pc:~$ kubectl apply -f <(cat <<EOF
apiVersion: v1
kind: Service
metadata:
  name: cluster-ip-svc
  namespace: only-cluster-ip-policy-ns
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
EOF
)
```

エラーになり更新できませんでした。

```
The services "cluster-ip-svc" is invalid: : ValidatingAdmissionPolicy 'only-cluster-ip-policy' with binding 'only-cluster-ip-policy-binding' denied request: failed expression: object.spec.type == 'ClusterIP'
```

```bash
hum@ryzen5pc:~$ kubectl get svc -n only-cluster-ip-policy-ns
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
cluster-ip-svc   ClusterIP   10.106.82.125   <none>        80/TCP    6m2s
```
