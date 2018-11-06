# Deploy SCF on AKS

## Deploy AKS
https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough

## Config Azure AKS
### Enable Swap Accounting for AKS nodes
az vm run-command invoke -g MC_thomasAKSCluster_thomasAKSCluster_southeastasia -n aks-nodepool1-94301426-0 --command-id RunShellScript \
   --scripts "sudo sed -i 's|linux.*./boot/vmlinuz-.*|& swapaccount=1|' /boot/grub/grub.cfg"

az vm restart -g MC_thomasAKSCluster_thomasAKSCluster_southeastasia -n aks-nodepool1-94301426-0

### Config Network
az network public-ip create \
 --resource-group MC_thomasAKSCluster_thomasAKSCluster_southeastasia\
 --name thomasAKSCluster-public-ip \
 --allocation-method Static

az network lb create \
 --resource-group MC_thomasAKSCluster_thomasAKSCluster_southeastasia\
 --name thomasAKSCluster-lb \
 --public-ip-address thomasAKSCluster-public-ip \
 --frontend-ip-name thomasAKSCluster-lb-front \
 --backend-pool-name thomasAKSCluster-lb-back

az network nic ip-config address-pool add \
    --resource-group MC_thomasAKSCluster_thomasAKSCluster_southeastasia\
    --nic-name aks-nodepool1-94301426-nic-0 \
    --ip-config-name ipconfig1 \
    --lb-name thomasAKSCluster-lb \
    --address-pool thomasAKSCluster-lb-back

export CAPPORTS="80 443 4443 2222 2793 8443"
export AKSNAME="thomasAKSCluster"

for i in $CAPPORTS
do
    az network lb probe create \
    --resource-group MC_thomasAKSCluster_thomasAKSCluster_southeastasia \
    --lb-name $AKSNAME-lb \
    --name probe-$i \
    --protocol tcp \
    --port $i

    az network lb rule create \
    --resource-group MC_thomasAKSCluster_thomasAKSCluster_southeastasia \
    --lb-name $AKSNAME-lb \
    --name rule-$i \
    --protocol Tcp \
    --frontend-ip-name $AKSNAME-lb-front \
    --backend-pool-name $AKSNAME-lb-back \
    --frontend-port $i \
    --backend-port $i \
    --probe probe-$i
done

### Deploy UAA
helm install suse/uaa \
--name susecf-uaa \
--namespace uaa \
--values scf-config-values.yaml

### Deploy SCF
SECRET=$(kubectl get pods --namespace uaa \
-o jsonpath='{.items[?(.metadata.name=="uaa-0")].spec.containers[?(.name=="uaa")].env[?(.name=="INTERNAL_CA_CERT")].valueFrom.secretKeyRef.name}')

CA_CERT="$(kubectl get secret $SECRET --namespace uaa \
-o jsonpath="{.data['internal-ca-cert']}" | base64 --decode -)"

helm install suse/cf \
--name susecf-scf \
--namespace scf \
--values scf-config-values.yaml \
--set "secrets.UAA_CA_CERT=${CA_CERT}"
