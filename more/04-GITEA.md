# Gitea deployment

Based on https://gitea.com/gitea/helm-chart

```bash
# start on node1
multipass shell node1

# provides storage
microk8s enable hostpath-storage
k get sc

# add helm repo and install gitea
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
helm install gitea gitea-charts/gitea -n gitea --create-namespace

# what we have got
k get all -n gitea
k get po --watch -n gitea

# check service
k get svc -n gitea


k describe svc gitea-http -n gitea

# expose gitea via ingress
export MYID=mko # use your own! 
cat << EOF |  sed "s/gitea.cloudguard.rocks/gitea-$MYID.cloudguard.rocks/" | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: gitea-ingress
 namespace: gitea
 annotations:
   cert-manager.io/cluster-issuer: lets-encrypt
   external-dns.alpha.kubernetes.io/hostname: gitea.cloudguard.rocks.
   external-dns.alpha.kubernetes.io/ttl: "60"
spec:
 ingressClassName: public
 tls:
 - hosts:
   - gitea.cloudguard.rocks
   secretName: gitea-ingress-tls
 rules:
 - host: gitea.cloudguard.rocks
   http:
     paths:
     - backend:
         service:
           name: gitea-http
           port:
             number: 3000
       path: /
       pathType: Prefix
EOF

# check ingress
k get ing -n gitea

# check cert - wait until it is ready
k get certificate -n gitea --watch

# visit Gitea at https://gitea-${MYID}.cloudguard.rocks
echo "Visit https://gitea-${MYID}.cloudguard.rocks"

# CLI test
curl -vvv https://gitea-${MYID}.cloudguard.rocks 2>&1
curl -vvv https://gitea-${MYID}.cloudguard.rocks 2>&1 | grep CN

# storage
k get pvc -n gitea

# uninstall
helm uninstall -n gitea gitea
k delete ns gitea

# storage
k logs -f -n kube-system deploy/hostpath-provisioner
k get pod -n kube-system -o wide --show-labels
k logs -f -n kube-system $(k get pod -n kube-system -o name -l k8s-app=hostpath-provisioner)
k get pvc -A
k get pv -A
```