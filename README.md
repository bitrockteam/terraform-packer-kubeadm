Provision Kubernetes with kubeadm + Terraform + Packer
=================


[![See in Action](https://lawofattractionsolutions.com/wp-content/uploads/2016/04/action-clapboard.png)](http://www.youtube.com/watch?v=J8RGm0rBAIg "Kubernetes on DO")



## Requirements:


* Packer
* Terraform



## Bake the kubernetes snapshot: 

```bash
export DIGITALOCEAN_API_TOKEN="<your_DO_token>"

cd terraform-packer-kubeadm

packer build -machine-readable \
    packer-kubernetes.json \
    | tee packer-kubernetes.log
```

Create a key pair:

```bash
ssh-keygen -t rsa -P "" -f ./k8s-key
```

Create the master node:

```bash
export TF_VAR_token=$DIGITALOCEAN_API_TOKEN

export TF_VAR_k8s_snapshot_id=$(grep \
    'artifact,0,id' \
    packer-kubernetes.log \
    | cut -d: -f2)

terraform init

terraform plan \
    -target="digitalocean_droplet.k8s_master" \
    -out plan

terraform apply plan
```

Once the master node is up and running. You can Provision and Join the two nodes to the cluster:

```bash
ssh-keyscan \
    $(terraform output master-ip) \
    >> ~/.ssh/known_hosts

export TF_VAR_k8s_join_command=$(ssh \
    -i k8s-key \
    root@$(terraform output master-ip) \
    "kubeadm token create --print-join-command")

terraform plan -out plan

terraform apply plan
```

Log into your new K8s cluster:

```bash
ssh -i k8s-key \
    root@$(terraform output master-ip)

export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl get nodes
```

```
NAME         STATUS     ROLES     AGE       VERSION
k8s-master   NotReady   master    5m        v1.9.7
k8s-node-1   NotReady   <none>    1m        v1.9.7
k8s-node-2   NotReady   <none>    1m        v1.9.7
```

```bash
exit

ssh -i k8s-key \
    root@$(terraform output master-ip) \
    "cat /etc/kubernetes/admin.conf" \
    | tee admin.conf

export KUBECONFIG=admin.conf

kubectl get nodes
```

## Install calico for CNI

```bash
kubectl apply -f https://docs.projectcalico.org/v2.1/getting-started/kubernetes/installation/hosted/kubeadm/1.6/calico.yaml

```
```
kubectl get nodes

NAME         STATUS    ROLES     AGE       VERSION
k8s-master   Ready     master    5m        v1.9.7
k8s-node-1   Ready     <none>    2m        v1.9.7
k8s-node-2   Ready     <none>    2m        v1.9.7
```

## To teardown the cluster

```bash
terraform destroy --force
```
