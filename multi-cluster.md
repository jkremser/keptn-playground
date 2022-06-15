## start ctrl cluster

```bash
# k3d cluster create keptn-ctrl -p "8082:80@loadbalancer" --k3s-arg "--kube-proxy-arg=conntrack-max-per-core=0@server:*" --agents 1
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
    --set remoteControlPlane.api.hostname=${KEPTN_ENDPOINT} \
    --set remoteControlPlane.api.apiValidateTls=false \
    --set remoteControlPlane.api.token=${KEPTN_API_TOKEN}
done
```

