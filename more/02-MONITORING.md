# Add cluster monitoring with Prometheus and Grafana

```shell
# on node1
multipass shell node1

# add helm repo
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

# deploy monitoring stack into NS monitoring
helm install prom-operator-01 prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# wait for all pods to be ready
kubectl get po -n monitoring --watch

# expose grafana service via ingress (you may limit access to your IP with AppSec)
export MYID=mko # use your own! 
cat << EOF |  sed "s/mon.cloudguard.rocks/mon-$MYID.cloudguard.rocks/" | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: grafana-ingress
 namespace: monitoring
 annotations:
   cert-manager.io/cluster-issuer: lets-encrypt
   external-dns.alpha.kubernetes.io/hostname: mon.cloudguard.rocks.
   external-dns.alpha.kubernetes.io/ttl: "60"
spec:
 ingressClassName: public
 tls:
 - hosts:
   - mon.cloudguard.rocks
   secretName: grafana-ingress-tls
 rules:
 - host: mon.cloudguard.rocks
   http:
     paths:
     - backend:
         service:
           name: prom-operator-01-grafana
           port:
             number: 80
       path: /
       pathType: Prefix
EOF

# check ingress
k describe ingress -n monitoring

# wait for cert-manager to issue certificate
k get certificate -n monitoring --watch

# check DNS record
dig +short @1.1.1.1 mon-$MYID.cloudguard.rocks
echo dig +short @1.1.1.1 mon-$MYID.cloudguard.rocks
watch -d dig +short @1.1.1.1 mon-$MYID.cloudguard.rocks

# visit Grafana at https://mon-${MYID}.cloudguard.rocks
echo "Visit https://mon-${MYID}.cloudguard.rocks"

# CLI test
curl -vvv https://mon-${MYID}.cloudguard.rocks 2>&1 
curl -vvv https://mon-${MYID}.cloudguard.rocks 2>&1 | grep CN

# troubleshooting
k logs -f  -n cert-manager deploy/cert-manager
k logs -n cert-manager deploy/cert-manager | grep "mon-${MYID}.cloudguard.rocks"

k logs -f -n external-dns deploy/external-dns
k logs -n external-dns deploy/external-dns | grep "mon-${MYID}.cloudguard.rocks"

# password
k get secret/prom-operator-01-grafana -n monitoring -o json | jq -r '.data["admin-password"]' | base64 -d; echo
```