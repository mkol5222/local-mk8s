# Publishing web apps with effortless DNS and TLS - local MicroK8s cluster

Launch first cluster node with cloud-init
```shell
multipass launch -v -n node1 --cloud-init cloud-init.yml -m 4G -d 10G -c 4

multipass shell node1
```

Check Hyper-V Default Virtual Switch address range
```shell
Get-NetIPAddress -InterfaceAlias "vEthernet (Default Switch)" | Select-Object IPAddress, PrefixLength
# e.g. 172.18.160.1/20 on host machine NIC
```


In node1 shell:
```shell
# wait for cluster to get ready
microk8s status -w

k get nodes -o wide
k get pods -o wide -A

# MetalLB setup
microk8s enable metallb:172.18.160.128-172.18.160.199

# try it out
k create deploy web --image nginx --replicas 3
k expose deploy web --port 80 --target-port 80 --type LoadBalancer
k get svc web
# 
# NAME   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
# web    LoadBalancer   10.152.183.181   172.18.160.128   80:32625/TCP   2s

# test it on host machine
# http://172.18.160.128

# install k9s tui
ark get k9s
sudo mv /home/ubuntu/.arkade/bin/k9s /usr/local/bin/

# it depends on kube config
mkdir ~/.kube; sudo microk8s config > ~/.kube/config
chmod o= ~/.kube/config
chmod g= ~/.kube/config

k9s 
```

External DNS setup
```shell
k create ns external-dns

# use real Cloudflare API token and e-mail - ask instructor
export EMAIL=someone@example.com
export CFTOKEN='bring-your-own-token'

kubectl -n external-dns create secret generic  cf-dns-setup --from-literal=CF_API_EMAIL=$EMAIL --from-literal=CF_API_TOKEN=$CFTOKEN

cat << 'EOF' | k apply -n external-dns -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services","endpoints","pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions","networking.k8s.io"]
  resources: ["ingresses"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.k8s.io/external-dns/external-dns:v0.14.0
        args:
        - --source=service 
        - --source=ingress 
        - --domain-filter=cloudguard.rocks 
        - --provider=cloudflare
        env:
        - name: CF_API_TOKEN
          valueFrom:
             secretKeyRef:
                name: cf-dns-setup
                key: CF_API_TOKEN
        - name: CF_API_EMAIL
          valueFrom:
             secretKeyRef:
                name: cf-dns-setup
                key: CF_API_EMAIL
---
EOF

k get all -n external-dns

# create A record for web service
k get svc web
# unique id - e.g. mko for Martin Koldovsky to assure uniqueness of resources like DNS records
export MYID=mko
k annotate service web "external-dns.alpha.kubernetes.io/hostname=web-${MYID}.cloudguard.rocks."
k annotate service web "external-dns.alpha.kubernetes.io/ttl=60" 
k describe svc web # check annotations
k logs -f -n external-dns deploy/external-dns

dig +short @1.1.1.1 web-${MYID}.cloudguard.rocks
echo "Visit http://web-${MYID}.cloudguard.rocks from host machine browser"

# summary we have DNS record for web service, 
# but we would love to have also HTTPS access with valid certificate
```

Cert manager cluster issuer with DNS-01 validator
```shell

# use real Cloudflare API token and e-mail - ask instructor
export EMAIL=someone@example.com
export CFTOKEN='bring-your-own-token'

k -n cert-manager create secret generic  cloudflare-api-token-secret --from-literal=api-token=$CFTOKEN

cat << 'EOF' | sed "s/someone@example.com/$EMAIL/" | k apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
 name: lets-encrypt
 namespace: cert-manager
spec:
 acme:
   email: someone@example.com
   server: https://acme-v02.api.letsencrypt.org/directory
   privateKeySecretRef:
     # Secret resource that will be used to store the account's private key.
     name: lets-encrypt-priviate-key
   # Add a single challenge solver, DNS01 using cloudflare
   solvers:
    - dns01:
        cloudflare:
          email: someone@example.com
          apiTokenSecretRef:
            name: cloudflare-api-token-secret
            key: api-token
EOF
```

Ingress setup
```shell
# will replace default MicroK8s ingress controller with NGINX
microk8s disable ingress

ark get kubectl
ark get helm
echo 'export PATH=$PATH:/home/ubuntu/.arkade/bin' >> ~/.bashrc
source ~/.bashrc

ark install ingress-nginx --namespace ingress --set controller.ingressClassResource.name=public

# verify external IP of ingress controller
k get svc -n ingress

# create ingress resource
k get svc web

export MYID=mko
dig +short @1.1.1.1 web-${MYID}.cloudguard.rocks

cat << 'EOF' |  sed "s/www.cloudguard.rocks/www-$MYID.cloudguard.rocks/" | k apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: web
 annotations:
   cert-manager.io/cluster-issuer: lets-encrypt
   external-dns.alpha.kubernetes.io/hostname: www-${MYID}.cloudguard.rocks.
   external-dns.alpha.kubernetes.io/ttl: "60"
spec:
 ingressClassName: public
 tls:
 - hosts:
   - www.cloudguard.rocks
   secretName: appsec-ingress-tls
 rules:
 - host: www.cloudguard.rocks
   http:
     paths:
     - backend:
         service:
           name: web
           port:
             number: 80
       path: /
       pathType: Prefix
EOF

dig +short @1.1.1.1 www-${MYID}.cloudguard.rocks
k describe ing/web

echo "Visit https://www-${MYID}.cloudguard.rocks from host machine browser"

k logs -f -n cert-manager deploy/cert-manager
```

Second cluster node
```shell
multipass launch -v -n node2 --cloud-init cloud-init.yml -m 4G -d 10G -c 4
# and join to cluster

```
