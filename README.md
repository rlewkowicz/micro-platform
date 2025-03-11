# Lab Setup

As simple as I've made this look, this will actually give you a fairly enterprise grade quick start for a phenomenal skeleton to a number of Kubernetes architectures. I’ll continue to grow this.


You can't ask ChatGPT how to make a nested enterprise k8s architecture lol. I don't think you'll find such a terse quick start covering so many bases anywhere else. I'm almost worried it makes it look trivial. This is phenomenally terse. It's an entire micro platform on one page. This is my skillset, an ability to condense worlds of information into one page. To navigate a decade of noise and put it all right in front of you.

### Base OS
Ubuntu 22.04.4 LTS x64 on bare metal with processor virtualization enabled (You can probably use 20.04/24.04 but ymmv)

### Docker 
```
curl -fsSL https://get.docker.com | sed 's/sleep [0-9]*/sleep 0/g' | sh
sudo curl -SL https://github.com/docker/compose/releases/download/v2.33.1/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

### Kubectl
```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### K3S
```
curl -sfL https://get.k3s.io | K3S_KUBECONFIG_MODE="644" INSTALL_K3S_EXEC="--disable-network-policy --disable=traefik --node-name=control-plane" sh -
export KUBECONFIG=/etc/rancher/k3s/k3s.yaml
```

### KubeVirt
```
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)

echo $VERSION
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-operator.yaml"
kubectl apply -f "https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/kubevirt-cr.yaml"
export VERSION=$(basename $(curl -s -w %{redirect_url} https://github.com/kubevirt/containerized-data-importer/releases/latest))
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-operator.yaml
kubectl apply -f https://github.com/kubevirt/containerized-data-importer/releases/download/$VERSION/cdi-cr.yaml
kubectl patch cdi cdi -n cdi --type=merge -p '{"spec":{"config":{"filesystemOverhead":{"global":"0.2"}}}}'
export VERSION=$(curl -s https://storage.googleapis.com/kubevirt-prow/release/kubevirt/kubevirt/stable.txt)

wget https://github.com/kubevirt/kubevirt/releases/download/${VERSION}/virtctl-${VERSION}-linux-amd64
mv virtctl-${VERSION}-linux-amd64 virtctl && chmod 700 virtctl
sudo install virtctl /usr/local/bin

(exit 1)
while [ $? != 0 ]; do
printf "\nWaiting for Kubevirt to be ready...\n"
sleep 3
kubectl get kubevirt.kubevirt.io/kubevirt -n kubevirt -o=jsonpath="{.status.phase}" | grep Deployed
done
echo
```

### Local Path Prov Fix for CDI
```
kubectl delete storageclass local-path
kubectl apply -f manifests/local-path-storage.yaml
```

<!-- ### Multus Networking
```
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset-thick.yml
``` -->

<!-- ### CDI import the Ubuntu image
```
cat <<EOF > dv_ubuntu.yml
apiVersion: cdi.kubevirt.io/v1beta1
kind: DataVolume
metadata:
  name: "ubuntu"
spec:
  pvc:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 10Gi
  source:
    http:
      url: "https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img"
EOF

kubectl create -f dv_ubuntu.yml

(exit 1)
while [ $? != 0 ]; do
printf "\nWaiting for PVC's to come online...\n"
sleep 3
kubectl get pvc | grep ubuntu-scratch | grep Bound
done
echo

printf "\nWaiting for CDI to import disk image...\n"
(exit 1)
while [ $? != 0 ]; do
kubectl get pods | grep importer-ubuntu | grep Running
done

kubectl logs -f importer-ubuntu | stdbuf -o0 grep -m 1 "Import Complete"
``` -->

### Launch the Kubernetes VM Pool
```
kubectl apply -f manifests/kubernetes_vm_pool.yaml
```

## Halftime

At this point, you have a basic kubernetes cluster that supports Kubevirt, and 3 fullblooded vms that will actually house our distributed k8s cluster. From here we're going to install k8s (yes it's nested) on those vms. There's a whole world of information that is beyond the scope of this document regarding VM provisioning on k8s that is worth exploring.

You can reset the lab at any point by running:
```
kubectl delete virtualmachinepool k8s
```

And then redeploying the VM pool.

## Ansible and Kubespray
```
docker run --rm -it -d --name ansible --net=host \
--mount type=bind,source="$(pwd)"/ansible/inventory,dst=/inventory \
--mount type=bind,source="$(which virtctl)",dst=/bin/virtctl \
--mount type=bind,source="/etc/rancher/k3s/k3s.yaml",dst=/root/.kube/config \
--mount type=bind,source="$(pwd)"/ansible/ansible.cfg,dst=/kubespray/ansible.cfg \
--mount type=bind,source="$(pwd)"/virtctl-proxy-config,dst=/virtctl-proxy-config \
quay.io/kubespray/kubespray:v2.27.0 tail -f /dev/null

docker exec -ti ansible sh -c "pip install kubernetes && ansible-galaxy collection install kubevirt.core"
docker exec -ti ansible ansible-playbook -i /inventory cluster.yml -b -v
docker exec -ti ansible ansible -i /inventory all -m fetch -a "src=/etc/kubernetes/admin.conf dest=/inventory/ flat=yes" > /dev/null
sudo chown $USER ansible/inventory/admin.conf
IP=$(docker exec -ti ansible ansible -i /inventory all -m shell -a "ip -4 addr show enp1s0 | grep inet | awk '{print \$2}' | cut -d/ -f1" | tail -n 1 | sed 's/\x1B\[[0-9;]*[mK]//g' | grep -oP '\S*')
sed -i "s#server: https://127.0.0.1:6443#server: https://$IP:6443#g" ansible/inventory/admin.conf
```

