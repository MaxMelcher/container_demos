# Common issues

## New version of azure cli does not work with az aks browse or aks get credentials

'ManagedCluster' object has no attribute 'properties'

Downgrade azure cli to working version

```
WORKING_VERSION=2.0.23-1
sudo apt-get install azure-cli=$WORKING_VERSION
```

## Edit using NANO
```
KUBE_EDITOR="nano" kubectl edit svc/nginxpod-service
```

## NIC in failed state
reset nic via cli
```
az network nic update -g MC_* -n aks-nodepool1-*-nic-0
```

## Check if your cluster has RBAC
```
kubectl cluster-info dump --namespace kube-system | grep authorization-mode
```

## Find out source ip

```
curl ipinfo.io/ip

kubectl run azure-function-on-kubernetes --image=denniszielke/az-functions --port=80 --requests=cpu=100m

kubectl expose deployment azure-function-on-kubernetes --type=LoadBalancer

kubectl autoscale deploy azure-function-on-kubernetes --cpu-percent=20 --max=10 --min=1

apk add --update curl
export num=0 && while true; do curl -s -w "$num = %{time_namelookup}" "time nslookup google.com"; echo ""; num=$((num+1)); done

kubectl run alp --image=alpine:3.6

kubectl run alp --image=quay.io/collectai/alpine-curl

docker run --rm jess/curl -sSL ipinfo.io/ip
```

## Steps how to attach public IP to a worker node

find out the resource group that AKS created for the node VMs

    az group list -o table

list resources in the group and find the VM you want to access

    az resource list -g MC_kubernetes_kubernetes-cluster_ukwest -o table

show parameters of that VM, see for example: "adminUsername": "azureuser"

    az vm show -g aks-default-10832236-0 -n aks-default-10832236-0

create the public IP

    az network public-ip create -g MC_kub_ter_k_m_iuasdf_iuasdf_westeurope -n test-ip

find out correct NIC where to add the public IP

    az network nic list -g MC_kub_ter_k_m_iuasdf_iuasdf_westeurope -o table

find out the name of the ipconfig within that NIC

    az network nic ip-config list --nic-name aks-default-10832236-nic-0 -g MC_kub_ter_k_m_iuasdf_iuasdf_westeurope

modify the ipconfig by adding the public IP address

    az network nic ip-config update -g MC_kub_ter_k_m_iuasdf_iuasdf_westeurope --nic-name aks-default-10832236-nic-0 --name ipconfig1 --public-ip-address test-ip

find out what the allocated public IP address is

    az network public-ip show -g MC_kubernetes_kubernetes-cluster_ukwest -n test-ip

then finally connect with SSH

    ssh azureuser@<public ip address>

```
cat <<EOF | kubectl apply -f -
kind: Pod
apiVersion: v1
metadata:
  name: curl
spec:
  containers:
    - name: curl
      image: jess/curl
      args: ["-sSL", "ipinfo.io/ip"]
EOF

kubectl logs curl
```

kubectl exec runclient -- bash -c "date && \
      echo 1 && \
      echo 2"


kubectl exec alpclient -- bash -c "var=1 && \
    while true ; do && \
      res=$( { curl -o /dev/null -s -w %{time_namelookup}\\\\n  http://www.google.com; } 2>&1 ) && \
      var=$((var+1)) && \
      if [[ $res =~ ^[1-9] ]]; then && \
        now=$(date +'%T') && \
        echo '$var slow: $res $now' && \
        break && \
      fi && \
    done"

kubectl get pods |grep runclient|cut -f1 -d\  |\
while read pod; \
 do echo "$pod writing:";\
  kubectl exec -t $pod -- bash -c \
 var=1 \
while true ; do \
  res=$( { curl -o /dev/null -s -w %{time_namelookup}\\n  http://www.google.com; } 2>&1 ) \
  var=$((var+1)) \
  if [[ $res =~ ^[1-9] ]]; then \
    now=$(date +"%T") \
    echo "$var slow: $res $now" \
    break \
  fi \
done \
done

var=1;
while true ; do
  res=$( { curl -o /dev/null -s -w %{time_namelookup}\\n  http://nginx; } 2>&1 )
  var=$((var+1))
  now=$(date +"%T")
  echo "$var slow: $res $now"
  if [[ $res =~ ^[1-9] ]]; then
    now=$(date +"%T")
    echo "$var slow: $res $now"
    break
  fi
done

var=1;
while true ; do
  res=$( { curl -o /dev/null -s -w %{time_namelookup}\\n  http://www.google.com; } 2>&1 )
  var=$((var+1))
  now=$(date +"%T")
  echo "$var slow: $res $now"
done

for i in `seq 1 1000`; do time curl -s google.com > /dev/null; done

kubectl run -i --tty busybox --image=busybox --restart=Never -- sh   


scp /path/to/file username@a:/path/to/destination


scp username@b:/path/to/file /path/to/destination

scp dennis@40.114.247.218:/var/log/azure/cluster-provision.log cluster-provision.log
scp dennis@40.114.247.218:/var/log/cloud-init-output.log cloud-init-output.log
