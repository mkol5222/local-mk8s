# Replace Kubernetes NGINX Ingress Controller with CloudGuard WAF

```shell
# start on node1
multipass shell node1

# assume ark installed nginx-ingress
kubectl get all -n ingress
# handling class public
kubectl get ingress -A
kubectl describe ingressclass -n ingress

# helm deplpyment
helm ls -n ingress

# now we know how to uninstall
helm uninstall -n ingress ingress-nginx
helm ls -n ingress

# get agent token from CloudGuard WAF Kubernetes profile 
# https://portal.checkpoint.com/dashboard/appsec#/waf-policy/profiles/

# autnetication token from previous step
export CPTOKEN=cp-06bc...

# install cloudguard waf
# install appsec ingress with helm chart from:
wget https://downloads.openappsec.io/packages/helm-charts/nginx-ingress/open-appsec-k8s-nginx-ingress-latest.tgz

# remember to set CPTOKEN above!
helm install appsec open-appsec-k8s-nginx-ingress-latest.tgz \
--set controller.appsec.mode=managed --set appsec.agentToken=$CPTOKEN \
--set appsec.image.registry="" \
--set appsec.image.repository=checkpoint \
--set appsec.image.image=infinity-next-nano-agent \
--set appsec.image.tag=831851 \
--set controller.hostNetwork=true \
--set controller.ingressClass=public \
--set controller.ingressClassResource.name=public \
--set controller.ingressClassResource.controllerValue="k8s.io/public" \
--set appsec.persistence.enabled=false \
--set controller.service.externalTrafficPolicy=Local \
-n appsec --create-namespace

# now appsec ingress is deployed to NS appsec - wait once it is ready
kubectl get po -n appsec --watch

# also visible in portal - list of connected agents
# https://portal.checkpoint.com/dashboard/appsec#/waf-policy/agents?status=Connected

# previously configured ingress objects should be now protected by CloudGuard WAF
export MYID=mko # use your own!
dig +short @1.1.1.1 www-${MYID}.cloudguard.rocks
sudo resolvectl flush-caches
curl -vvv https://www-${MYID}.cloudguard.rocks 2>&1 

# remember to add asset representing your site in CloudGuard WAF
# https://portal.checkpoint.com/dashboard/appsec#/waf-policy/assets

# once asset is in place, new policy enforced by agent. lets cause some incidents
curl -vvv "https://www-${MYID}.cloudguard.rocks/?param=cat+../../etc/passwd" 2>&1
curl -vvv "https://www-${MYID}.cloudguard.rocks/?z=UNION+1=1" 2>&1

# check at monitoring
# https://portal.checkpoint.com/dashboard/appsec#/waf-monitor/high-and-critical-wlc/
```


Reference: dedicated notes at https://github.com/mkol5222/chkp-public-notes/blob/master/appsec/ingress/appsec-ingress-azurevm.md