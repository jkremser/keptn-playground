## start k3s cluster

```bash
k3d cluster create mykeptn -p "8082:80@loadbalancer" --k3s-arg "--kube-proxy-arg=conntrack-max-per-core=0@server:*" --agents 1
```

## install keptn cli

```bash
curl -sL https://get.keptn.sh | bash
```

## install keptn to the k8s cluster

```bash
keptn install --use-case=continuous-delivery
```

## configure ingress

```bash
curl -SL https://raw.githubusercontent.com/keptn/examples/master/quickstart/expose-keptn.sh | bash
```

## create sample podtato project

```bash
./multistage-delivery.sh
```


# change the deployment

```bash
k edit deployments.apps job-executor-service
```

`PUBSUB_UR`L value `nats://keptn-nats-cluster -> nats://keptn-nats` on `"distributor"` container
and remove the liveness probe on the same container