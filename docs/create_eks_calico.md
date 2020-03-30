
### Calico Installation

Once cluster is setup, install calico using below command:

```
kubectl apply -f https://raw.githubusercontent.com/aws/amazon-vpc-cni-k8s/master/config/v1.3/calico.yaml
```

Validate that daemon sets came online

```
kubectl get daemonset calico-node --namespace kube-system
```

It should give below outcome

```
NAME          DESIRED   CURRENT   READY     UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
calico-node   3         3         3         3            3           <none>          38s

```

To delete Calico from EKS clsuer

```
kubectl delete daemonset calico-node --namespace kubesystem
```

Output:

```
daemonset.extensions "calico-node" deleted
```

### Create Network Policy and Demo:
https://docs.aws.amazon.com/eks/latest/userguide/calico.html

Below is standard network policy to cover most of use cases:

```
# Default deny all ingress
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: default-deny-ingress
  namespace: {{namespace}}
spec:
spec:
  podSelector: {}
  ingress: []

# Allow ingress to webapp 
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-ingress-webapp
  namespace: {{namespace}}
spec:
  podSelector:
    matchLabels:
      name: webapp
  ingress:
  - {}

# Allow ingress from web demo to other pods
---
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: internal-services
  namespace: {{namespace}}
spec:
  podSelector:
    matchExpressions:
     - {key: app, operator: In, values: [app1, app2]}
  ingress:
  - from:
    - podSelector:
        matchLabels:
          name: webapp
---

# Deny external egress and open only DNS traffic from specific pods
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
    name: deny-external-egress
    namespace: {{namespace}}
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - ports:
    - port: 53
      protocol: UDP
    - port: 53
      protocol: TCP
  - to:
    - namespaceSelector: {}

---
# Egress network policy to specific pods 

apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
    name: egress-network-policy
    namespace: {{namespace}}
spec:
  podSelector:
    matchExpressions:
     - {key: app, operator: In, values: [app1, app2]}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 172.48.190.180/32
    ports:
    - protocol: TCP
      port: 8098
    ports:
    - protocol: TCP
      port: 443
---

```


#### Reference
https://docs.aws.amazon.com/eks/latest/userguide/calico.html
