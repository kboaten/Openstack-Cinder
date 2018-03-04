# Openstack-Cinder - Enabling Cinder on Kubernetes Cluster

Highly based on this: 

https://stackoverflow.com/questions/46067591/how-to-use-openstack-cinder-to-create-storage-class-and-dynamically-provision-pe

https://medium.com/@arthur.souzamiranda/kubernetes-with-openstack-cloud-provider-current-state-and-upcoming-changes-part-1-of-2-48b161ea449a

Note: This was tested on Kubernetes 1.9 and Openstack Pike

#create a file on the host
vim /etc/kubernetes/cloud.conf 

[Global]
auth-url=http://<host>:5000/v3
username=<username>
password=<pwd>
region=RegionOne
tenant-name=kubernetes
domain-name=Default

[LoadBalancer]
subnet-id=<openstack subnet id>
floating-network-id=<public network id>


# Configuring Kubelet
This step is needed for all nodes, including the master.

#Run this on host to copy config to all nodes including master
scp -i /root/pcf.key  /etc/kubernetes/cloud.conf ubuntu@kube-master:
 
#login to kubernetes cluster master  node and run this command
ssh -i pcf.key ubuntu@<host>

sudo mv /home/ubuntu/cloud.conf /etc/kubernetes/
  
sudo vim /etc/kubernetes/kubelet.env
#then add the code below to KUBELET_ARGS

--cloud-provider=openstack \
--cloud-config=/etc/kubernetes/cloud.conf \

#now reload service
sudo systemctl daemon-reload
sudo systemctl restart kubelet

 
-------------------------------

# Configuring Master Nodes

sudo cp /etc/kubernetes/manifests/kube-controller-manager.manifest /etc/kubernetes/manifests/kube-controller-manager.yaml
sudo vim /etc/kubernetes/manifests/kube-controller-manager.yaml

#Add this under spec/containers/command
 
```yaml
[...] 
spec:
  containers:
  - command:
    - kube-controller-manager (kube-apiserver)
    - --cloud-provider=openstack
    - --cloud-config=/etc/kubernetes/cloud.conf
[...]
    volumeMounts:
    - mountPath: /etc/kubernetes/cloud.conf
      name: cloud-config
      readOnly: true 
[...]
 volumes:
  - hostPath:
      path: /etc/kubernetes/cloud.conf
      type: FileOrCreate
    name: cloud-config
[...]
```
### Verify that system picked up changes
kubectl describe pod kube-controller-manager -n kube-system | grep '/etc/kubernetes/cloud.conf'



# Create a default Storage Class
 
vi cinder-storage.yaml 
```yaml 
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: standard
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  labels:
    kubernetes.io/cluster-service: "true"
    addonmanager.kubernetes.io/mode: EnsureExists
provisioner: kubernetes.io/cinder
parameters:
  type: iscsi
  availability: nova
allowVolumeExpansion: true
``` 

kubectl create -f cinder-storage.yaml
