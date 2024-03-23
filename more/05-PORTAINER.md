# Portainer deployment with DNS and TLS

Based on https://portainer.github.io/k8s/charts/portainer/

```bash
# start on node1
multipass shell node1

# create namespace
kubectl create namespace portainer
# add helm repo and install portainer
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

# install portainer
helm upgrade -i -n portainer portainer portainer/portainer

# note port access
 export NODE_PORT=$(kubectl get --namespace portainer -o jsonpath="{.spec.ports[1].nodePort}" services portainer)
  export NODE_IP=$(kubectl get nodes --namespace portainer -o jsonpath="{.items[0].status.addresses[0].address}")
  echo https://$NODE_IP:$NODE_PORT

 # check the service
kubectl get svc -n portainer

# expose portainer via ingress
export MYID=mko # use your own!
cat << EOF |  sed "s/portainer.cloudguard.rocks/portainer-$MYID.cloudguard.rocks/" | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: portainer-ingress
 namespace: portainer
 annotations:
   cert-manager.io/cluster-issuer: lets-encrypt
   external-dns.alpha.kubernetes.io/hostname: portainer.cloudguard.rocks.
   external-dns.alpha.kubernetes.io/ttl: "60"
spec:
 ingressClassName: public
 tls:
 - hosts:
   - portainer.cloudguard.rocks
   secretName: portainer-ingress-tls
 rules:
 - host: portainer.cloudguard.rocks
   http:
     paths:
     - backend:
         service:
           name: portainer
           port:
             number: 9000
       path: /
       pathType: Prefix
EOF

# check ingress
k get ing -n portainer
k describe ing -n portainer

# wait for cert-manager to issue certificate
k get certificate -n portainer --watch

# cli test
curl -vvv https://portainer-${MYID}.cloudguard.rocks 2>&1
curl -vvv https://portainer-${MYID}.cloudguard.rocks 2>&1 | grep CN

# visit Portainer at https://portainer-${MYID}.cloudguard.rocks
echo "Visit https://portainer-${MYID}.cloudguard.rocks"

```