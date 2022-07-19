## start ctrl cluster

```bash
k3d cluster create -c k3d/keptn-ctrl.yaml
k3d cluster create -c k3d/keptn-exec.yaml
```
## install keptn ctrl plane

```bash
kubectl config use-context k3d-keptn-ctrl
keptn install -y
```

## configure ingress

```bash
curl -SL https://raw.githubusercontent.com/keptn/examples/master/quickstart/expose-keptn.sh | bash
```

## env vars

```bash
echo "webui: http://127.0.0.1.nip.io:8082"
export KEPTN_ENDPOINT=$(docker network inspect keptn-network -f '{{ (index .IPAM.Config 0).Gateway }}'):8082
export KEPTN_API_TOKEN=$(kubectl get secret keptn-api-token -n keptn --context=k3d-keptn-ctrl -ojsonpath={.data.keptn-api-token} | base64 --decode)
export JES_VERSION="0.2.3"
```

## install keptn exec plane

```bash
kubectl config use-context k3d-keptn-exec
for s in jmeter helm ; do
    echo Installing $s-service
    helm upgrade -i $s-service https://github.com/keptn/keptn/releases/download/0.16.0/$s-service-0.16.0.tgz \
    -n keptn-exec --create-namespace --kube-context=k3d-keptn-exec \
    --set remoteControlPlane.enabled=true \
    --set remoteControlPlane.api.protocol=http \
    --set remoteControlPlane.api.hostname="${KEPTN_ENDPOINT}" \
    --set remoteControlPlane.api.apiValidateTls=false \
    --set remoteControlPlane.api.token="${KEPTN_API_TOKEN}"
done
kubectl config use-context k3d-keptn-ctrl
```

# install job-executor-service
```bash
helm upgrade -i job-executor-service https://github.com/keptn-contrib/job-executor-service/releases/download/${JES_VERSION}/job-executor-service-${JES_VERSION}.tgz \
-n keptn-jes --create-namespace --kube-context=k3d-keptn-exec \
--set remoteControlPlane.topicSubscription=sh.keptn.event.cluster-switch.triggered \
--set remoteControlPlane.api.token="${KEPTN_API_TOKEN}" \
--set remoteControlPlane.api.hostname="${KEPTN_ENDPOINT}" \
--set remoteControlPlane.api.protocol=http \
--set jobConfig.serviceAccount.name=switcher \
--set jobConfig.serviceAccount.create=false

helm upgrade -i job-executor-service https://github.com/keptn-contrib/job-executor-service/releases/download/${JES_VERSION}/job-executor-service-${JES_VERSION}.tgz \
-n keptn-jes --create-namespace --kube-context=k3d-keptn-ctrl \
--set remoteControlPlane.topicSubscription=sh.keptn.event.cluster-switch.triggered \
--set remoteControlPlane.api.token="${KEPTN_API_TOKEN}" \
--set remoteControlPlane.api.hostname="${KEPTN_ENDPOINT}" \
--set remoteControlPlane.api.protocol=http \
--set jobConfig.serviceAccount.create=false
```

# create project and link it with git repo
keptn create project switch -y -s manifests/shipyard.yaml --git-user=jkremser --git-token=$GH_TOKEN --git-remote-url=https://github.com/jkremser/cluster-switch-action

# create service
keptn create service hello --project switch -y

# add the resource to the service
keptn add-resource --project switch --service hello --stage production --resource manifests/switch.yaml --resourceUri job/config.yaml


# rbac (cruding gslbs and deployments should be enough)
for c in exec ctrl ; do
    kubectl -n keptn-jes create sa switcher --dry-run=client -oyaml | kubectl apply --context k3d-keptn-$c -f -
    kubectl create clusterrolebinding switcher-crb --clusterrole cluster-admin --serviceaccount keptn-jes:switcher --dry-run=client -oyaml | kubectl apply --context k3d-keptn-$c -f -
done

# trigger the event
```bash
keptn trigger sequence cluster-switch --project switch --service hello --stage production
```

# check it in the web ui
```bash
_url=http://127.0.0.1.nip.io:8082/bridge/project/switch/sequence
which open && open $_url || xdg-open $_url
```

# and/or check the cluster switch in the terminal

```bash
kubectl get gslb ${gslb:-test-gslb-failover} -n ${gslb_ns:-test-gslb} -o=jsonpath='{.spec.strategy.primaryGeoTag}{"\n"}' -w
```