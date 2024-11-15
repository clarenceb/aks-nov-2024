# AKS Demos for November 2024

### Common steps

```sh
az extension add --name aks-preview || az extension update --name aks-preview

LOCATION=australiaeast
```

## Static Egress Gateway (via AKS add-on)

```sh
az feature register --namespace "Microsoft.ContainerService" --name "StaticEgressGatewayPreview"
az feature show --namespace "Microsoft.ContainerService" --name "StaticEgressGatewayPreview"
# Wait until the above shows as registered

az provider register --namespace Microsoft.ContainerService

STATIC_EGRESS_RG_NAME=aks-nov24-staticegress
STATIC_EGRESS_CLUSTER_NAME=aks-staticegress

# Create a cluster with static egress gateway
az group create --name $STATIC_EGRESS_RG_NAME --location $LOCATION
az aks create -n $STATIC_EGRESS_CLUSTER_NAME -g $STATIC_EGRESS_RG_NAME --enable-static-egress-gateway

# Create a Gateway Node pool
# The --gateway-prefix-size is the size of the public IP prefix to be applied to the gateway node pool nodes. The allowed range is 28-31.
az aks nodepool add \
    --cluster-name $STATIC_EGRESS_CLUSTER_NAME \
    --name staticegress \
    --resource-group $STATIC_EGRESS_RG_NAME \
    --mode gateway \
    --node-count 2 \
    --gateway-prefix-size 31

az aks get-credentials --resource-group $STATIC_EGRESS_RG_NAME --name $STATIC_EGRESS_CLUSTER_NAME

kubectl get pods -A

kubectl get node
# NAME                                   STATUS   ROLES    AGE    VERSION
# aks-nodepool1-28548646-vmss000000      Ready    <none>   10m    v1.29.9
# aks-nodepool1-28548646-vmss000001      Ready    <none>   10m    v1.29.9
# aks-nodepool1-28548646-vmss000002      Ready    <none>   10m    v1.29.9
# aks-staticegress-42573227-vmss000000   Ready    <none>   116s   v1.29.9
# aks-staticegress-42573227-vmss000001   Ready    <none>   115s   v1.29.9

kubectl apply -f static-gateway-config.yaml
kubectl get pods -A
kubectl describe StaticGatewayConfiguration mydemo-static-gateway-config

# Deploy a sample app without static egress config
kubectl apply -f egress-app1-no-gateway.yaml

# Deploy a sample app with static egress config
kubectl apply -f egress-app2-static-gateway.yaml

kubectl get pod

POD1=$(kubectl get pods -l app=app1 -o name)
kubectl exec -ti $POD1 -- bash
curl https://ifconfig.me -w "\n"
# 4.200.96.17

POD2=$(kubectl get pods -l app=app2 -o name)
kubectl exec -ti $POD2 -- bash
curl https://ifconfig.me -w "\n"
# 20.58.176.4
```

For private ip set up, see: https://github.com/clarenceb/static-egress-private-ip

- Adjust the sample to use the ASK add-on instead of the static egress gateway upstream helm chart

### Clean up:

```sh
kubectl delete -f egress-app1-no-gateway.yaml
kubectl appdeletely -f egress-app2-static-gateway.yaml
kubectl delete -f static-gateway-config.yaml

az aks nodepool delete --cluster-name $STATIC_EGRESS_CLUSTER_NAME -g $STATIC_EGRESS_RG_NAME -n staticegress
az aks update -n $STATIC_EGRESS_CLUSTER_NAME -g $STATIC_EGRESS_RG_NAME --disable-static-egress-gateway

az group delete --name $STATIC_EGRESS_RG_NAME  --yes --no-wait

az feature unregister --namespace "Microsoft.ContainerService" --name "StaticEgressGatewayPreview"
az provider register --namespace "Microsoft.ContainerService"
```

## FQDN filtering with ACNS

```sh
ACNS_RG_NAME=aks-nov24-fqdnfilter
ACNS_EGRESS_CLUSTER_NAME=aks-fqdnfilter

# Create an AKS cluster
az group create --name $ACNS_RG_NAME --location $LOCATION
az aks create \
    --name $ACNS_EGRESS_CLUSTER_NAME \
    --resource-group $ACNS_RG_NAME \
    --generate-ssh-keys \
    --location $LOCATION \
    --max-pods 250 \
    --network-plugin azure \
    --network-plugin-mode overlay \
    --network-dataplane cilium \
    --node-count 2 \
    --pod-cidr 192.168.0.0/16 \
    --kubernetes-version 1.29 \
    --enable-acns

az aks get-credentials --resource-group $ACNS_RG_NAME --name $ACNS_EGRESS_CLUSTER_NAME

kubectl get pods -A
kubectl get node

kubectl apply -f demo-fqdn-policy.yaml

kubectl run -it --rm client --image=k8s.gcr.io/e2e-test-images/agnhost:2.43 --labels="app=demo-container" --command -- bash

kubectl get pod --sort-by="{spec.nodeName}" -o wide
kubectl get pod -n kube-system -o wide --field-selector spec.nodeName="aks-nodepool1-20547176-vmss000000" | grep "cilium"

# cilium pod
kubectl exec -it -n kube-system cilium-b5jqw -- sh
cilium monitor -t drop
# xx drop (Policy denied) flow 0xfddd76f6 to endpoint 0, ifindex 29, file bpf_lxc.c:1274, , identity 48447->world: 192.168.0.149:45830 -> 93.184.215.14:80 tcp SYN

# demo pod
./agnhost connect www.bing.com:80
./agnhost connect www.microsoft.com:80

curl -t3 www.bing.com
curl -t3 www.microsoft.com
```

### Clean up:

```sh
kubectl delete -f egress-app1-no-gateway.yaml
kubectl appdeletely -f egress-app2-static-gateway.yaml
kubectl delete -f static-gateway-config.yaml

az aks nodepool delete --cluster-name <cluster-name> -n <nodepool-name>
az aks update -n <cluster-name> -g <resource-group> --disable-static-egress-gateway

az group delete --name $RG_NAME --yes --no-wait

az feature unregister --namespace "Microsoft.ContainerService" --name "StaticEgressGatewayPreview"
az provider register --namespace "Microsoft.ContainerService"
```

## Azure Linux 3.0

Create cluster with Azure Linux 3.0 and deploy a sample app:

```sh
az feature register --namespace "Microsoft.ContainerService" --name "AzureLinuxV3Preview"
az feature show --namespace "Microsoft.ContainerService" --name "AzureLinuxV3Preview"
# Wait until the above shows as registered

az provider register --namespace Microsoft.ContainerService

LINUX3_RG_NAME=aks-nov24-linux3
LINUX3_CLUSTER_NAME=aks-azure-linux3
# Check the latest version `az aks get-versions -l $LOCATION -o table`
# Use 1.31.1+ for Azure Linux 3.0
LINUX3_CLUSTER_VERSION=1.31.1

az group create --name $LINUX3_RG_NAME --location $LOCATION
az aks create --name $LINUX3_CLUSTER_NAME --resource-group $LINUX3_RG_NAME --os-sku AzureLinux --kubernetes-version $LINUX3_CLUSTER_VERSION

az aks get-credentials --resource-group $LINUX3_RG_NAME --name $LINUX3_CLUSTER_NAME

kubectl get pods -A

kubectl apply -f aks-store-quickstart.yaml

kubectl get pods
kubectl get services

# Start a Azure Linux based container
kubectl run -it busybox --image=mcr.microsoft.com/azurelinux/busybox:1.36
cat /etc/os-release

# SSH onto AKS node
kubectl krew install ssh-jump
kubectl get pod -o wide
# Note the node that the busybox pod is running on
kubectl ssh-jump aks-nodepool1-28557653-vmss000002 -u azureuser -i ~/.ssh/id_rsa
cat /etc/os-release
ctr ns ls
ctr -n k8s.io containers list
sudo crictl ps
# Get the container ID for busybox
sudo crictl exec -it 696af19969b82 cat /etc/os-release
tdnf list available -C 2>/dev/null | grep -E "x86_64|noarch" | wc
 -l
 # 4466
```

Check available container images on MCR:

[https://mcr.microsoft.com/en-us/catalog?search=azurelinux&type=exact&page=1](https://mcr.microsoft.com/en-us/catalog?search=azurelinux&type=exact&page=1)

### Clean up:

```sh
kubectl delete -f aks-store-quickstart.yaml
kubectl delete pod busybox

az group delete --name $LINUX3_RG_NAME --yes --no-wait

az feature unregister --namespace "Microsoft.ContainerService" --name "AzureLinuxV3Preview"
az provider register --namespace "Microsoft.ContainerService"
```
