# Deploy Keycloak with Ingress and TLS

based on https://www.keycloak.org/getting-started/getting-started-kube

```bash
# start on node1
multipass shell node1

# download keycloak.yaml
curl -OL https://raw.githubusercontent.com/keycloak/keycloak-quickstarts/latest/kubernetes/keycloak.yaml
# consider updating keycloack for new KEYCLOAK_ADMIN and KEYCLOAK_ADMIN_PASSWORD
vi keycloak.yaml

# deploy keycloak
k create ns keycloak
k apply -f keycloak.yaml -n keycloak

# what we ahev got
k get all -n keycloak

# focus on service
k describe svc -n keycloak

# expose keycloak via ingress
export MYID=mko # use your own! 
cat << EOF |  sed "s/keycloak.cloudguard.rocks/keycloak-$MYID.cloudguard.rocks/" | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 name: keycloak-ingress
 namespace: keycloak
 annotations:
   cert-manager.io/cluster-issuer: lets-encrypt
   external-dns.alpha.kubernetes.io/hostname: keycloak.cloudguard.rocks.
   external-dns.alpha.kubernetes.io/ttl: "60"
spec:
 ingressClassName: public
 tls:
 - hosts:
   - keycloak.cloudguard.rocks
   secretName: keycloak-ingress-tls
 rules:
 - host: keycloak.cloudguard.rocks
   http:
     paths:
     - backend:
         service:
           name: keycloak
           port:
             number: 8080
       path: /
       pathType: Prefix
EOF

# check ingress
k describe ingress -n keycloak
# check cert - wait until it is ready
k get certificate -n keycloak --watch
# check DNS record
dig +short @1.1.1.1 keycloak-$MYID.cloudguard.rocks
echo dig +short @1.1.1.1 keycloak-$MYID.cloudguard.rocks
watch -d dig +short @1.1.1.1 keycloak-$MYID.cloudguard.rocks

# visit Grafana at https://keycloak-${MYID}.cloudguard.rocks
echo "Visit https://keycloak-${MYID}.cloudguard.rocks"

# CLI test
curl -vvv https://keycloak-${MYID}.cloudguard.rocks 2>&1 
curl -vvv https://keycloak-${MYID}.cloudguard.rocks 2>&1 | grep CN

```
