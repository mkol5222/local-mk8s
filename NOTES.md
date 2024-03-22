# Publishing web apps with effortless DNS and TLS - local MicroK8s cluster

We are using [Multipass](https://multipass.run/) to launch Ubuntu LTS VMs on Windows 10 Hyper-V. We will install MicroK8s on the VMs and deploy a simple web app. We will use MetalLB for LoadBalancer service type, External-DNS for DNS records, Cert-Manager for TLS certificates, and NGINX Ingress Controller for Ingress resources.


### Launch first cluster node with cloud-init

```shell
# create VM
multipass launch -v -n node1 --cloud-init cloud-init.yml -m 4G -d 10G -c 4
# enter VM command line
multipass shell node1
```

### Check Hyper-V Default Virtual Switch address range

```shell
# on Windows host machine
Get-NetIPAddress -InterfaceAlias "vEthernet (Default Switch)" | Select-Object IPAddress, PrefixLength
# e.g. 172.18.160.1/20 on host machine NIC used for Hyper-V Default Switch
```


### Expose Web app using MetalLB provided IP

Back in node1 shell:

```shell
# wait for cluster to get ready
microk8s status -w

# check what we have got
k get nodes -o wide
k get pods -o wide -A

# MetalLB setup - external IP used to reach web apps (svc type LoadBalancer external IP)
microk8s enable metallb:172.18.160.128-172.18.160.199

# try it out

# make web server with 3 replicas
k create deploy web --image nginx --replicas 3
k get pods -l app=web -o wide

# expose it externally on IP provided by MetalLB
k expose deploy web --port 80 --target-port 80 --type LoadBalancer

# look for external IP
k get svc web

# 
# NAME   TYPE           CLUSTER-IP       EXTERNAL-IP      PORT(S)        AGE
# web    LoadBalancer   10.152.183.181   172.18.160.128   80:32625/TCP   2s

WEBIP=$(k get svc web -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Access http://$WEBIP on host machine browser"
```

### Install K9s using Arkade (ark)

`k9s` is a terminal UI to interact with Kubernetes clusters. Arkade (aka `ark`) is a CLI that makes installation of cloud native tools easier.

```shell

# install k9s tui
ark get k9s
sudo mv /home/ubuntu/.arkade/bin/k9s /usr/local/bin/

# it depends on kube config
mkdir ~/.kube; sudo microk8s config > ~/.kube/config
chmod o= ~/.kube/config
chmod g= ~/.kube/config

k9s 
```

### External DNS setup

Lets use External-DNS to create DNS records for our web services automatically.

```shell
# create namespace for external-dns
k create ns external-dns

# use real Cloudflare API token and e-mail - ask instructor
export EMAIL=someone@example.com
export CFTOKEN='bring-your-own-token'

# store as secret
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

# check what was deployed for you
k get all -n external-dns

# create A record for web service

# will use existing web service
k get svc web

# unique id - e.g. mko for Martin Koldovsky to assure uniqueness of resources like DNS records
export MYID=mko # use your own!

# DNS record is created based on service or ingress annotations
k annotate service web "external-dns.alpha.kubernetes.io/hostname=web-${MYID}.cloudguard.rocks."
k annotate service web "external-dns.alpha.kubernetes.io/ttl=60" 

# review annotations
k describe svc web # check annotations

# check DNS record creation process
k logs -f -n external-dns deploy/external-dns

# once record is created
dig +short @1.1.1.1 web-${MYID}.cloudguard.rocks

# now service is reacheble also by DNS name (not only IP)
echo "Visit http://web-${MYID}.cloudguard.rocks from host machine browser"

# summary we have DNS record for web service, 
# but we would love to have also HTTPS access with valid certificate
```

### Cert manager cluster issuer with DNS-01 validator

Lets use Cert-Manager to get TLS certificates for our web services using Lets Encrypt. We provide Cloudflare API token to Cert-Manager to create DNS records for DNS-01 challenge.

```shell

# use real Cloudflare API token and e-mail - ask instructor
export EMAIL=someone@example.com
export CFTOKEN='bring-your-own-token'

# store as secret
k -n cert-manager create secret generic  cloudflare-api-token-secret --from-literal=api-token=$CFTOKEN

# make cluster issuer for lets-encrypt with DNS-01 challenge
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

### Ingress setup

Once cert-manager is ready to monitor ingress annotations for TLS certificates, we can create Ingress resources for our web services.

```shell
# will replace default MicroK8s ingress controller with NGINX
microk8s disable ingress

# install NGINX Ingress Controller - kubectl binary required
ark get kubectl
ark get helm
echo 'export PATH=$PATH:/home/ubuntu/.arkade/bin' >> ~/.bashrc
source ~/.bashrc

# install ingress-nginx using Arkade tool into ingress namespace
# notice that we are using "public" ingress class
ark install ingress-nginx --namespace ingress --set controller.ingressClassResource.name=public

# verify external IP of ingress controller
k get svc -n ingress

# verify existing web service that we are about to publish
k get svc web

# unique id - e.g. mko for Martin Koldovsky to assure uniqueness of resources like DNS records
export MYID=mko # use your own!
# direct service access was:
dig +short @1.1.1.1 web-${MYID}.cloudguard.rocks

# create ingress resource for web service
# will publish on https://www-${MYID}.cloudguard.rocks

cat << EOF |  sed "s/www.cloudguard.rocks/www-$MYID.cloudguard.rocks/" | k apply -f -
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
   secretName: web-ingress-tls
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

# check what was created
k describe ing/web

# was DNS A record created?
dig +short @1.1.1.1 www-${MYID}.cloudguard.rocks

# was certificate provided? ready state?
k get certificate

# test it out
echo "Visit https://www-${MYID}.cloudguard.rocks from host machine browser"

# CLI test
sudo resolvectl flush-caches
curl -vvv https://www-${MYID}.cloudguard.rocks 2>&1 

# review process of TLS cert creation (DNS-01 challenge)
k logs -f -n cert-manager deploy/cert-manager
```

### Second cluster node

```shell
# add one more VM
multipass launch -v -n node2 --cloud-init cloud-init.yml -m 4G -d 10G -c 4
# login and wait for microk8s to be ready
multipass shell node2
sudo microk8s status -w

# other console - login to node1
multipass shell node1
# get command for node2 on node1 console
sudo microk8s add-node
# something like: microk8s join 172.18.161.168:25000/6d082e36ca9959d58ca759f2d55f26be/804826265516 --worker

# back to node2
multipass shell node2
# see abobe!!!
sudo microk8s join 172.18.161.168:25000/6d082e36ca9959d58ca759f2d55f26be/804826265516 --worker

# back to node1
multipass shell node1
k get nodes -o wide --watch
# expect: node1 and node2 ready
# NAME    STATUS   ROLES    AGE    VERSION   INTERNAL-IP      EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION       CONTAINER-RUNTIME
# node1   Ready    <none>   138m   v1.28.7   172.18.161.168   <none>        Ubuntu 22.04.4 LTS   5.15.0-101-generic   containerd://1.6.28
# node2   Ready    <none>   32s    v1.28.7   172.18.172.162   <none>        Ubuntu 22.04.4 LTS   5.15.0-101-generic   containerd://1.6.28

# how are pods distributed?
k get pods -o wide -A
# node2 pods
k get pods -o wide -A | grep node2

# redistribute web pods
k scale deploy web --replicas=1
# new will choose according resources available
k scale deploy web --replicas=5
# check distribution
k get pods -l app=web -o wide

# MetalLB owner VIP for Ingress - still same
k get svc -n ingress
# pointed to by DNS
export MYID=mko # use your own!
dig +short @1.1.1.1 www-${MYID}.cloudguard.rocks
```

### Replace general Ingress with CloudGuard WAF (AppSec) Ingress

```shell
# work in progress...
```

### Cleanup

You might want to delete VMs with cluster nodes to release cpu, ram, and disk space.

```shell
multipass delete -p node1 node2
```