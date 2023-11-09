# KinD: Linkerd multi-cluster LAB


## Requirements

- Linux OS
- [Docker](https://docs.docker.com/)
- [KinD](https://kind.sigs.k8s.io/)
- [kubectl](https://kubernetes.io/docs/reference/kubectl/)
- [helm](https://helm.sh/docs/intro/install/)
- [step](https://smallstep.com/docs/step-cli/installation)
- [linkerd](https://linkerd.io/2.13/getting-started/#step-1-install-the-cli)
- [linkerd-smi](https://linkerd.io/2.13/tasks/linkerd-smi/#cli)

```
### linkerd & linkerd-smi CLI install:
curl --proto '=https' --tlsv1.2 -sSfL https://run.linkerd.io/install | sh
curl -sL https://linkerd.github.io/linkerd-smi/install | sh
export PATH=$PATH:/home/davar/.linkerd2/bin
```


### Setup k8s clusters

Run the `setup-clusters.sh` script. It creates three KinD clusters:

- One primary cluster (`primary`)
- Two remotes (`remote1`, `remote2`)

`kubectl` contexts are named respectively:

- `kind-primary`
- `kind-remote1`
- `kind-remote2`

```
Example Output:

[+] Creating KinD clusters
   ⠿ [remote2] Cluster created
   ⠿ [remote1] Cluster created
   ⠿ [primary] Cluster created
[+] Adding routes to other clusters
   ⠿ [primary] Route to 10.20.0.0/24 added
   ⠿ [primary] Route to 10.30.0.0/24 added
   ⠿ [remote1] Route to 10.10.0.0/24 added
   ⠿ [remote1] Route to 10.30.0.0/24 added
   ⠿ [remote2] Route to 10.10.0.0/24 added
   ⠿ [remote2] Route to 10.20.0.0/24 added
[+] Deploying MetalLB inside primary
   ⠿ [primary] MetalLB deployed
[+] Deploying MetalLB inside clusters
   ⠿ [primary] MetalLB deployed
   ⠿ [remote1] MetalLB deployed
   ⠿ [remote2] MetalLB deployed
```

### (Alternative) KinD cluster and MetalLB setup not using setup-clusters.sh :

```
Note: kind-primary.yaml & kind-remote1.yaml & kind-remote2.yaml ---> apiServerAddress: 192.168.3.2 # PUT YOUR IP ADDRESSS OF YOUR MACHINE HERE DUMMIE! ;-) 

kind create cluster --config kind-primary.yaml
kind create cluster --config kind-remote1.yaml
kind create cluster --config kind-remote2.yaml


helm repo add metallb https://metallb.github.io/metallb --force-update
for ctx in kind-primary kind-remote1 kind-remote2; do
 helm install metallb -n metallb-system --create-namespace metallb/metallb --kube-context=${ctx}
done

docker network inspect kind -f '{{.IPAM.Config}}'

Example Output:
docker network inspect kind -f '{{.IPAM.Config}}'
[{172.18.0.0/16  172.18.0.1 map[]} {fc00:f853:ccd:e793::/64   map[]}]


kubectl --context=kind-primary apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.0.61-172.18.0.70

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  labels:
    todos: verdade
  name: example
  namespace: metallb-system
EOF

kubectl --context=kind-remote1 apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.0.71-172.18.0.80

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  labels:
    todos: verdade
  name: example
  namespace: metallb-system
EOF

kubectl --context=kind-remote2 apply -f - <<EOF
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 172.18.0.81-172.18.0.90

---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  labels:
    todos: verdade
  name: example
  namespace: metallb-system
EOF
```

### Setup Linkerd multi-cluster

```bash
kubectl config get-contexts 
CURRENT   NAME            CLUSTER         AUTHINFO        NAMESPACE
          kind-primary   kind-primary   kind-primary   
          kind-remote1    kind-remote1    kind-remote1    
*         kind-remote2    kind-remote2    kind-remote2  
```

#### Create Certificates
* We like to use the step CLI to generate these certificates. If you prefer openssl instead, feel free to use that! To generate the trust anchor with step, you can run:
* https://smallstep.com/cli/
* https://linkerd.io/2.14/tasks/multicluster/


```bash
mkdir certs && cd certs
step certificate create \
  root.linkerd.cluster.local \
  ca.crt ca.key --profile root-ca \
  --no-password --insecure
step certificate create \
  identity.linkerd.cluster.local \
  issuer.crt issuer.key \
  --profile intermediate-ca \
  --not-after 8760h --no-password \
  --insecure --ca ca.crt --ca-key ca.key

alias lk='linkerd'
alias ka='kubectl apply -f '

for ctx in kind-primary kind-remote1 kind-remote2; do                   
  echo "install crd ${ctx}"
  linkerd install --context=${ctx} --crds | kubectl apply -f - --context=${ctx};

  echo "install linkerd ${ctx}";
  linkerd install --context=${ctx} \
    --identity-trust-anchors-file=ca.crt \
    --identity-issuer-certificate-file=issuer.crt \
    --identity-issuer-key-file=issuer.key | kubectl apply -f - --context=${ctx};

  echo "install viz ${ctx}";
  linkerd --context=${ctx} viz install | kubectl apply -f - --context=${ctx};

  echo "install multicluster ${ctx}";    
  linkerd --context=${ctx} multicluster install | kubectl apply -f - --context=${ctx};

  echo "install smi ${ctx}";        
  linkerd smi install --context=${ctx}  | kubectl apply -f - --context=${ctx};
done
```

```bash
for ctx in kind-primary kind-remote1 kind-remote2; do
  printf "Checking cluster: ${ctx} ........."
  while [ "$(kubectl --context=${ctx} -n linkerd-multicluster get service linkerd-gateway -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers)" = "<none>" ]; do
      printf '.'
      sleep 1
  done
  echo "`kubectl --context=${ctx} -n linkerd-multicluster get service linkerd-gateway -o 'custom-columns=:.status.loadBalancer.ingress[0].ip' --no-headers`"
  printf "\n"
done
```

### Example Output:
Checking cluster: kind-primary .........172.17.255.10

Checking cluster: kind-remote1 .........172.17.255.30

Checking cluster: kind-remote2 .........172.17.255.50

---------------Comment(no execute) 

Error: probe-gateway-kind-primary.linkerd-multicluster mirrored from cluster [kind-primary] has no endpoints--------------
It is an issue with the way that Kind sets up access to the API server. You need to use the --api-server-address to tell Linkerd how to access the API server. There is an example here: https://github.com/olix0r/l2-k3d-multi/blob/master/link.sh

# Unfortunately, the credentials have the API server IP as addressed from
# localhost and not the docker network, so we have to patch that up.

Se o contexto é o primário, você deve coletar o endpoint da API do cluster primário e adicionar no comando.
No remote1 e remote2 também, consequentemente.
Comando:
```bash
k get endpoints -n default --context=kind-primary
k get endpoints -n default --context=kind-remote1
k get endpoints -n default --context=kind-remote2
```


```bash
 linkerd --context=kind-primary multicluster link --cluster-name kind-primary --api-server-address="172.18.0.3:6443" | kubectl apply -f - --context=kind-remote1
 linkerd --context=kind-remote1 multicluster link --cluster-name kind-remote1 --api-server-address="172.18.0.7:6443" | kubectl apply -f - --context=kind-primary
 linkerd --context=kind-primary multicluster link --cluster-name kind-primary --api-server-address="172.18.0.3:6443" | kubectl apply -f - --context=kind-remote2
 linkerd --context=kind-remote2 multicluster link --cluster-name kind-remote2 --api-server-address="172.18.0.9:6443" | kubectl apply -f - --context=kind-primary

```
---------------
Comment(no execute) 

Error: probe-gateway-kind-primary.linkerd-multicluster mirrored from cluster [kind-primary] has no endpoints

--------------

### Check

```bash
for ctx in kind-primary kind-remote1 kind-remote2; do
  echo "Checking link....${ctx}"
  linkerd --context=${ctx} multicluster check

  echo "Checking gateways ...${ctx}"
  linkerd --context=${ctx} multicluster gateways

  echo "..............done ${ctx}"
done
```

Example Output:
Checking link....kind-primary
linkerd-multicluster
--------------------
√ Link CRD exists
√ multicluster extension proxies are healthy
√ multicluster extension proxies are up-to-date
√ multicluster extension proxies and cli versions match

Status check results are √
Checking gateways ...kind-primary
CLUSTER  ALIVE    NUM_SVC      LATENCY  
..............done kind-primary
Checking link....kind-remote1
linkerd-multicluster
--------------------
√ Link CRD exists
√ Link resources are valid
	* kind-primary
√ remote cluster access credentials are valid
	* kind-primary
√ clusters share trust anchors
	* kind-primary
√ service mirror controller has required permissions
	* kind-primary
√ service mirror controllers are running
	* kind-primary
√ probe services able to communicate with all gateway mirrors
	* kind-primary
√ multicluster extension proxies are healthy
√ multicluster extension proxies are up-to-date
√ multicluster extension proxies and cli versions match

Status check results are √
Checking gateways ...kind-remote1
CLUSTER       ALIVE    NUM_SVC      LATENCY  
kind-primary  True           0          2ms  
..............done kind-remote1
Checking link....kind-remote2
linkerd-multicluster
--------------------
√ Link CRD exists
√ Link resources are valid
	* kind-primary
√ remote cluster access credentials are valid
	* kind-primary
√ clusters share trust anchors
	* kind-primary
√ service mirror controller has required permissions
	* kind-primary
√ service mirror controllers are running
	* kind-primary
√ probe services able to communicate with all gateway mirrors
	* kind-primary
√ multicluster extension proxies are healthy
√ multicluster extension proxies are up-to-date
√ multicluster extension proxies and cli versions match

Status check results are √
Checking gateways ...kind-remote2
CLUSTER       ALIVE    NUM_SVC      LATENCY  
kind-primary  True           0          1ms  
..............done kind-remote2

### Get endpoint for --api-server-address
➜ docker inspect primary-control-plane | grep IPAddress

    "IPAddress": "172.18.0.2"

➜ docker inspect primary-control-plane | grep IPAddress

    "IPAddress": "172.18.0.3",

➜ docker inspect remote1-control-plane | grep IPAddress

    "IPAddress": "172.18.0.7",

➜ docker inspect remote2-control-plane | grep IPAddress

    "IPAddress": "172.18.0.9",


### Adicionar os serviços de testes nos clusters criados:
```bash
for ctx in kind-primary kind-remote1 kind-remote2; do
  echo "Adding test services on cluster: ${ctx} ........."
  kubectl --context=${ctx} apply \
    -n test -k "github.com/Vinaum8/website/multicluster/${ctx}/"
  kubectl --context=${ctx} -n test \
    rollout status deploy/podinfo || break
  echo "-------------"
done
```

Checar o status dos pods no cluster:
```bash
for ctx in kind-primary kind-remote1 kind-remote2; do
  echo "Check pods on cluster: ${ctx} ........."
  kubectl get pod -n test --context=${ctx}
done
```

kubectl port-forward -n test svc/frontend 8080:8080 --context=kind-primary
Browser: http://localhost:8080

kubectl port-forward -n test svc/frontend 8080:8080 --context=kind-remote1
Browser: http://localhost:8080

```bash
for ctx in kind-primary kind-remote1 kind-remote2; do
  echo -en "\n\nLabel svc podinfo on cluster: ${ctx} .........\n"
  kubectl label svc -n test podinfo mirror.linkerd.io/exported=true --context=${ctx}
  sleep 4

  echo "Check services transConnected (if word exists )....on cluster ${ctx}"
  kubectl get svc -n test --context=${ctx}
done
```

### Check and Traffic Split
```bash 
kubectl get svc --context=kind-primary -n linkerd-multicluster linkerd-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

kubectl get svc --context=kind-remote1 -n linkerd-multicluster linkerd-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

kubectl get svc --context=kind-remote2 -n linkerd-multicluster linkerd-gateway -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
``` 

```bash 
kubectl get endpoints --context=kind-remote1 -n test podinfo -o jsonpath='{.subsets[*].addresses[*].ip}'

kubectl get endpoints --context=kind-primary -n test podinfo -o jsonpath='{.subsets[*].addresses[*].ip}'
```


```bash
kubectl --context=kind-primary -n test exec -c nginx -it \
  $(kubectl --context=kind-primary -n test get po -l app=frontend \
    --no-headers -o custom-columns=:.metadata.name) \
  -- /bin/sh -c "apk add curl && curl http://podinfo-kind-remote1:9898"
```


kubectl --context=kind-primary apply -f - <<EOF
apiVersion: split.smi-spec.io/v1alpha1
kind: TrafficSplit
metadata:
  name: podinfo
  namespace: test
spec:
  service: podinfo
  backends:
  - service: podinfo
    weight: 40
  - service: podinfo-kind-remote1
    weight: 30
  - service: podinfo-kind-remote2
    weight: 30
EOF


kubectl get endpoints --context=kind-remote1 -n test podinfo-kind-primary -o jsonpath='{.subsets[*].addresses[*].ip}'
kubectl get endpoints --context=kind-primary -n test podinfo-kind-remote1 -o jsonpath='{.subsets[*].addresses[*].ip}'


$ kubectl port-forward -n test --context=kind-primary svc/frontend 8080


### Helm - Nginx Ingress
```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx --force-update
for ctx in kind-primary kind-remote1 kind-remote2; do
  echo "Install ingress-nginx - ${ctx}"
  helm install ingress-nginx -n ingress-nginx --create-namespace ingress-nginx/ingress-nginx --set controller.podAnnotations.linkerd.io/inject=enabled --kube-context=${ctx}
  
  echo "Wait for ingress-nginx installation to finish, 30s, wide"
  kubectl rollout status deploy -n ingress-nginx ingress-nginx-controller --context=${ctx}

  echo "Breeding an income for the podinfo - ${ctx}"
  kubectl --context=${ctx} -n test create ingress frontend --class nginx --rule="frontend-${ctx}.192.168.3.2.nip.io/*=frontend:8080" --annotation "nginx.ingress.kubernetes.io/service-upstream=true"
done
```

```bash
for ctx in kind-primary kind-remote1 kind-remote2; do
  echo "Add the hostname and IP to your /etc/hosts and wait for ingress-nginx to assign IP to the ingress, about 30s"
  kubectl --context=${ctx} get ingress -n test 
done
```

```bash 
grep front /etc/hosts
```
172.17.255.11   frontend-kind-primary.192.168.3.2.nip.io   
172.17.255.31   frontend-kind-remote1.192.168.3.2.nip.io   
172.17.255.51   frontend-kind-remote2.192.168.3.2.nip.io  

Browser: frontend-primary.192.168.3.2.nip.io

## Clean local environment
```
$ kind delete cluster --name=primary
Deleting cluster "primary" ...
$ kind delete cluster --name=remote1
Deleting cluster "remote1" ...
$ kind delete cluster --name=remote2
Deleting cluster "remote2" .
```

### Reference:
- Linkerd: https://linkerd.io
- https://linkerd.io/2.14/tasks/multicluster/#linking-the-clusters
- ServiceMesh: https://smi-spec.io/
- Credits: https://github.com/adonaicosta/linkerd-multicluster & https://github.com/olix0r/l2-k3d-multi/
