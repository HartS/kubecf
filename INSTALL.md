# Install Kubecf release

## Prerequisites

### Diego

### Eirini

- Kube CA trusted on nodes

 |             | v1.12 | v1.13 | v1.14 | v1.15 | v1.16 |
 | ----------- | ----- | ----- | ----- | ----- | ----- |
 | Google GKE  |       |       |       |       |       |
 | Amazon EKS  |       |       |       |       |       |
 | SUSE CaaSP  |       |       |       |       |       |
 | IBM Bluemix |       |       |       |       |       |
 | Azure AKS   |       |       |       |       |       |
 | Minikube    |  ✔     |       |       |       |       |
 | Kind        |  ✔   |       |       |       |       |


## Prepare the cluster

### minikube

We need more disk space in minikube, otherwise pods will get evicted and the deployment will be in a constant loop.

Create the cluster with:

    minikube start --memory=12000mb --cpus=4 --disk-size=40gb --kubernetes-version v1.14.1

### GKE

At least for Diego we need a node OS with XFS support.
The `--image-type UBUNTU` selects an OS with XFS support.

Create the cluster like this:

```
project=kubecf
clustername=kubecf-test
gcloud beta container --project "$project" clusters create "$clustername" \
      --zone "europe-west4-a" --no-enable-basic-auth --cluster-version "1.12.8-gke.10" \
      --machine-type "custom-6-13312" --image-type "UBUNTU" --disk-type "pd-standard" \
      --disk-size "100" --metadata disable-legacy-endpoints=true \
      --scopes "https://www.googleapis.com/auth/devstorage.read_only","https://www.googleapis.com/auth/logging.write","https://www.googleapis.com/auth/monitoring","https://www.googleapis.com/auth/servicecontrol","https://www.googleapis.com/auth/service.management.readonly","https://www.googleapis.com/auth/trace.append" \
      --preemptible --num-nodes "1" --enable-cloud-logging --enable-cloud-monitoring \
      --enable-ip-alias --network "projects/$project/global/networks/default" \
      --subnetwork "projects/$project/regions/europe-west4/subnetworks/default" \
      --default-max-pods-per-node "110" --addons HorizontalPodAutoscaling,HttpLoadBalancing \
      --enable-autoupgrade --enable-autorepair --no-shielded-integrity-monitoring
```

## Install Helm

Install Helm with RBAC, this involves creating the role first:

```
kubectl create -f <( cat <<EOF
apiVersion: v1
kind: ServiceAccount
metadata:
  name: tiller
  namespace: kube-system
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: tiller
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
  - kind: ServiceAccount
    name: tiller
    namespace: kube-system
EOF
)

helm init --upgrade --service-account tiller --wait
```

## Install CF-Operator

CF-Operator can be installed in a separate namespace:

```
helm install --namespace cfo --name cf-operator --set "operator.watchNamespace=kubecf" https://s3.amazonaws.com/cf-operators/helm-charts/cf-operator-v0.4.1%2B92.g77e53fda.tgz
```

This allows us to restart the operator, because it is not affected by webhooks. We can also delete the Kubecf deployment namespace to start from scratch, without redeploying the operator.

## Install Kubecf

Enable Eirini explicitly when installing. The `system_domain` DNS record needs to point to the IP of the external load balancer.

```
helm install --namespace kubecf --name kubecf https://scf-v3.s3.amazonaws.com/scf-3.0.0-82165ef3.tgz --set "system_domain=kubecf.suse.dev" --set "features.eirini=true"
```

## Expose Kubecf

### GKE

Make the CF router available via a load balancer:

```
kubectl expose service -n kubecf kubecf-router-v1-0 --type=LoadBalancer --name=kubecf-router-lb
```

The load balancer's public IP should have these DNS records:

```
app1.kubecf.suse.dev
app2.kubecf.suse.dev
app3.kubecf.suse.dev
login.kubecf.suse.dev
api.kubecf.suse.dev
uaa.kubecf.suse.dev
doppler.kubecf.suse.dev
log-stream.kubecf.suse.dev
```

If you are testing locally and have no control over a DNS zone, you can enter host aliases in your `/etc/hosts`:

```
192.168.99.112 app1.kubecf.suse.dev app2.kubecf.suse.dev app3.kubecf.suse.dev login.kubecf.suse.dev api.kubecf.suse.dev uaa.kubecf.suse.dev doppler.kubecf.suse.dev log-stream.kubecf.suse.dev
```