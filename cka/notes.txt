# Namespace / Service
- A pod can access a service in its own namespace by just using service name.
  A pod can access a service in a different namespace by using below format.
  <svc-name>.<ns-name>.<svc>.<cluster.local>
  e.g db-service.dev.svc.cluster.local
- When a service is created, a DNS entry is added automatically in this format:
  db-service.dev.svc.cluster.local
  Here db-service is service name, dev is namespace, svc represents service and cluster.local is domain.


# Service
- A service (any type of service) can be accessed by other pods inside the cluster using CLUSTER-IP or service name and port (service port)
  The IP of service is CLUSTER-IP. So CLUSTER-IP field is present for all service types.
- Service is created for target pods which you want to access.
- To access an application exposed through node port service:
  http://(ip of any of the nodes in cluster):nodePort
  Please note that even if application is not deployed on that node, 
  it can still be accessed from outside the cluster by giving ip of that node and nodePort.
- When we do kubectl get svc for a service of type Cluster IP, then the port it shows is port of service and not target port.
- When we do kubectl get svc for a service of type NodePort, then the ports it show are port of service and node port.

# Imperative vs Declarative
- Except kubectl apply, all are imperative commands.
- When you run kubectl apply, then local configuration file, live object configuration and last applied configuration stored under 
  metadata->annotations in json format are compared.
  
# 
- kubectl get po --no-headers
- kubectl edit will only work for properties in a pod which can be changed. e.g image, tolerations, activeDeadlineSeconds. 
  Else it wouldn't allow you to make the changes.


# Assigning Pods to Nodes
- Node Affinity and Node Selector does the same thing.
  Node Affinity is for handling complex requirements.
  affinity, nodeSelector and tolerations are properties in pod.
  For nodeSelector and affinity to work, labels need to be set on the nodes.
- Taints need to be set on nodes and Tolerations on pods.
- Taints and Tolerations prevent other pods from being placed on our nodes. (repel)
  Node Affinity and Node Selector prevent our pods from being placed on other nodes. (attract)
- For commands, refer CKAD notes.
- Use Exists (as operator) whether in tolerations or affinity if key alone in label matters and not value.

# DaemonSet 
- DaemonSet specification is very much similar to ReplicaSet.
  Just like replicaset, daemonset also makes use of labels and selectors.
   kubectl get ds
   kubectl describe ds <name>
   
- There is a hack for creating DaemonSet using imperative commands.
  Create a deployment yaml and then replace kind: Deployment with kind: DaemonSet and remove replicas and strategy from the deployment yaml.
  kubectl create deploy nginx --image=nginx --dry-run=client -o yaml
  
- Use cases for daemonset are kube-proxy, pod network add ons (weave-net)
   
# Static Pods
- To find static pods in a cluster, run the command kubectl get po -A and look for those with (nodename) appended in the name.
- To find the path of directory holding static pod definition files, run the command ps -ef | grep kubelet and identify the config file --config=/var/lib/kubelet/config.yaml. Then check in the config file for staticPodPath.
- To create a static pod, create a pod definition file and place it in directory holding the static pod definition files.
- To delete a static pod, identify the node on which static pod is running, ssh to that node and delete the static pod definition file.
- To make changes to static pod say image, make changes to static pod definition file. That should recreate the pod.
  If this doesn't work, then remove the file and create a new static pod definition file.
- You can create static pods directly on worker nodes by placing pod definition file in /etc/kubernetes/manifests.
  
# Deployments
- While changing the deployment strategy from Rolling Update to Recreate, make sure to delete the properties of Rolling Update as well, set at 'strategy.rollingUpdate'.
- With Deployments you can easily edit any field/property of the POD template. 
  Since the pod template is a child of the deployment specification,  with every change the deployment will automatically delete and create a new pod with the new changes. 
  So if you are asked to edit a property of a POD part of a deployment, you may do that simply by running the command
  kubectl edit deploy <deploy-name>

# Multi-Container Pods
- Containers share the same network space which means they can refer to each other as local host and they have access to the same storage volumes.
  This way you do not have to establish volume sharing or services between the containers to enable communication between them to create a multi container pod.
  
# Init Containers
- If there is more than one init container then they run in sequence.

# Cluster Maintenance (for instance applying OS patches)
- To Empty the node of all applications and mark it unschedulable
  kubectl drain node01 --ignore-daemonsets
  kubectl drain node02 --ignore-daemonsets --force
- Why force option is needed. Without force option, you can't delete Pods not managed by ReplicaSet, Job, DaemonSet or StatefulSet (use --force to override)
  If force option is applied, then standalone pods are lost forever.
- Mark the node as unschedulable only but do not remove any apps currently running on it .
  kubectl cordon node01
- To Configure the node to be schedulable again
  kubectl uncordon node01

  
# Cluster Upgrade Process
Note: Skipping MINOR versions when upgrading is unsupported.
- To find latest stable version available for upgrade
   kubeadm upgrade plan
- Drain the master of the workloads and mark it UnSchedulable 
   kubectl drain controlplane --ignore-daemonsets
- Upgrade kubeadm tool
   apt-get update
   apt-get install kubeadm=1.19.0-00 or apt-get install kubeadm=1.19.0*
   kubeadm version
- Upgrade master components
   kubeadm upgrade apply v1.19.0
- Upgrade the kubelet and kubectl on master node
   apt-get update
   apt-get install kubelet=1.19.0-00 kubectl=1.19.0-00 
- Restart the kubelet
   systemctl daemon-reload
   systemctl restart kubelet
- Mark the master/controlplane node as "Schedulable" again
   

- Drain the worker node of the workloads and mark it UnSchedulable (kubectl always runs on master)
   kubectl drain node01 --ignore-daemonsets
- Upgrade kubeadm tool on worker nodes
   ssh node01
   apt-get update
   apt-get install kubeadm=1.19.0-00
- Upgrade the worker node 
   kubeadm upgrade node
- Upgrade the kubelet and kubectl on worker node
   ssh node01
   apt-get update
   apt-get install kubelet=1.19.0-00 kubectl=1.19.0-00
- Restart the kubelet
   systemctl daemon-reload
   systemctl restart kubelet
- Mark the worker node as schedulable again (kubectl always run on master)
   kubectl uncordon node01
   
Note: When we run kubectl get no, in the version column, we get the version of kubelet running on the node.
      You can also run kubelet --version.
- Below command will give you kubernetes control plane version (server) and kubectl version (client)
  kubectl version
- kubeadm version

   
# ETCD
- kubectl exec etcd-minikube -n kube-system -- sh -c "etcdctl get / --prefix --keys-only --cacert /var/lib/minikube/certs/etcd/ca.crt --cert /var/lib/minikube/certs/etcd/server.crt --key /var/lib/minikube/certs/etcd/server.key"
- Configuration:
  advertise-client-urls - This is the address on which ETCD listens. This is the URL that should be configured on the kube-api server when it tries to reach the etcd server.
  listen-client-urls - Same as above.
  initial-cluster - The initial-cluster option is where you must specify the different instances of the ETCD service.
  cert-file - location of server certificate file.
  trusted-ca-file - location of CA certificate file.
  
- You can check for options either in etcd yaml file or in pod. 
- To check the version of ETCD running on the cluster.
  kubectl describe po etcd-controlplane -n kube-system
- The nodes in etcd cluster communicate internally on port 2380.
  api server reaches etcd cluster on port 2379.
 
# Backup and Restore method

- Take a snapshot of the ETCD database using the built-in snapshot functionality
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key snapshot save /opt/snapshot-pre-boot.db
- Restore ETCD Snapshot to a new data directory
   ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 --cacert=/etc/kubernetes/pki/etcd/ca.crt --cert=/etc/kubernetes/pki/etcd/server.crt --key=/etc/kubernetes/pki/etcd/server.key --data-dir=/var/lib/etcd-from-backup snapshot restore /opt/snapshot-pre-boot.db
- Update /etc/kubernetes/manifests/etcd.yaml
- Useful links:
   https://shahbhargav.medium.com/kubernetes-cluster-backup-and-restore-using-etcdctl-tool-35831702ab7e
   https://github.com/mmumshad/kubernetes-the-hard-way/blob/master/practice-questions-answers/cluster-maintenance/backup-etcd/etcd-backup-and-restore.md
   
   
# Storage
- What would happen to PV if PVC was deleted ?
  If PV's reclaim policy is Retain: PV is not deleted but PV is not available. PV shows the name of deleted PVC under claim column. This PV doesn't bind to another PVC. 
  If PV's reclaim policy is Recycle: PV is scrubbed. PV's status changes to Available and PV shows nothing under claim column. This PV should bind to another PVC (to be checked).
  If PV's reclaim policy is Delete: PV is deleted as well.
  
- When a PV is created, its status is available. When it is bound to PVC, its status changes to bound.
  When a PVC is created, its status is pending. When it is bound to PV, its status changes to bound.

- Once the pod is deleted, then only PVC can be deleted. Else not.
  Once PVC is deleted then based on PV's retention policy, PV will be handled.

- kubectl get sc
  https://kubernetes.io/docs/concepts/storage/storage-classes/#local
  
- The storage classes that don't use any provisioner (no-provisioner) don't support dynamic provisioning of volumes.

- Create a PV and PVC and check if they get bind even if POD is not scheduled.
  Yes, they bind even if PVC is not being used in a Pod.
  But if there is a storage class and if storage class makes use of VolumeBindingMode set to WaitForFirstConsumer then this will delay the binding and provisioning of PV until a Pod using the PVC is created.
  
- Only dynamically provisioned pvc can be resized and the storageclass that provisions the pvc must support resize.

# Pod Networking
- CoreDNS by default listens on port 53 which is the default port for DNS server.
  CoreDNS is deployed as two pods for redundancy, as part of a deployment. 
  The configuration file for CoreDNS is Corefile.
  Corefile is passed into the CoreDNS pod as a configMap object named as coredns.
- When we deploy CoreDNS solution, it also creates a service to make it available to other components within a cluster.
  The service is named as kube-dns by default. The IP address of this service is configured as nameserver on the PODs.
  Once the pods are configured with the right nameserver, you can now resolve other pods and services.
- eth0 is interface, veth0 is virtual interface. Interfaces have IP addresses.
- Useful commands: 
  netstat -nplt
  netstat -anp | grep etcd
- To identify the network plugin configured for Kubernetes
  ps -ef | grep kubelet
- The CNI plugin is configured in the kubelet service on each node in the cluster.
- To find out the range of IP addresses configured for PODs on the cluster.
  kubectl logs <weave-pod-name> weave -n kube-system | grep -i "ipalloc-range"
  or
  ip addr show weave
- To find out the IP Range configured for services within the cluster.
  cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep service-cluster-ip-range
- To find out what type of proxy is the kube-proxy configured to use.
  Check the logs of kube-proxy pod.
- To lookup service (from inside another pod):
  nslookup <svc name>.<ns name>.svc.cluster.local
  To lookup pod (from inside another pod)
  nslokup <pod name>.<ns name>.pod.cluster.local
  <pod name> would be xx-xx-xx-xx (replace dots in ip address with dashes).
 
- To find what network range are the nodes in the cluster part of,
  Run the command ip addr and look at the IP address assigned to the ens3 interfaces. 

  
# Installation
- Install kubeadm on master and worker nodes
- For initializing the control plane
  kubeadm init
- kubeadm tool doesn't deploy kubelets on worker nodes or master node.
- Using kubeadm tool, kube-proxy is deployed as daemonset.

# Scheduling
- For manual scheduling, set nodeName property in pod specification file.
  nodeName is a property for pods under spec.
  
- While configuring custom scheduler, Set below properties:
  name:<>
  leader-elect: false
  
  Add below options:
  --scheduler-name=<>
  --secure-port=<>
  
- For using custom scheduler, set schedulerName property under spec for pod.

# Troubleshooting
- Application failure
  - Check that name of objects in kubernetes cluster like pods, services etc are correct (as per specification).
  - Check that ports (node port, port, target port) on service are as per specification.
  - Check if service has discovered end points for the pod. If not, check that selectors configured on the service match with labels on pods.
  - Check the status of pod and number of restarts.
  - If pod is getting restarted, then check pod logs using -f option.
  - If pod is part of deployment then check deployment for specifications first and then pod.
  - If pod logs say that some file doesn't exist but file does exist on node then check volumes and volume mounts.
  - Check that values of environment variables in pod are as per specification.

- Control Plane failure
  - Check node status
  - Check application pods
  - Check control plane pods
  - If problem is with scaling of replicas in replicaset or deployment then check controller manager.
  - If pod is not getting scheduled (in pending state) then check scheduler.
  - If control plane components are not running as pods, then check status of services as

	service kube-apiserver status
	service kube-controller-manager status
	service kube-scheduler status
	service kube-proxy status

- Worker Node failure
  - Check the status of nodes (kubectl get no, kubectl describe no <>)
  - Check for cpu, memory and disk space on the node
  - journalctl -u kubelet      -- To check service logs
  - service kubelet start|stop|status|restart
  - ps -ef | grep kubelet
  - /etc/kubernetes/kubelet.conf has details about api server.
    If ip or port of api server is incorrect in this file, then kubelet wouldn't be able to connect to api server.
  - /var/lib/kubelet/config.yaml has details mainly about the kubelet.
  - If things are not good on worker node then check the status of kubelet, cni plugin and kube-proxy.
    If kube-proxy is running as pod (daemonset) then check its status on master.
	If cni plugin is running as pod (daemonset) then check its status on master.

- kubectl cluster-info
  Above command shows on which ip and port, kube-api server is running.
  Default kube-api server port is 6443.
  Default kube-controller-manager port is 10252.
  Default kube-scheduler port is 10251.
  Default kubelet port is 10250.
  2379 is the port of ETCD to which all control plane components connect to.
  2380 is only for etcd peer-to-peer connectivity when you have multiple master nodes. 

- Network failure
  - Check if kube-proxy daemonset and CNI plugin daemonset (weave-net) are running. If yes, check their logs. 
  - To edit kube-proxy daemonset
    kubectl edit ds kube-proxy -n kube-system
  - If you find CoreDNS pods in pending state, first check network plugin is installed.
  - Once a Pod network has been installed, you can confirm that it is working by checking that the CoreDNS Pod is Running.
  - If CoreDNS pods and the kube-dns service is working fine, check that kube-dns service has valid endpoints.
      kubectl -n kube-system get ep kube-dns
    If there are no endpoints for the service, inspect the service and make sure it uses the correct selectors and ports.


# RBAC
- To identify the authorization modes configured on the cluster
  kubectl describe po kube-apiserver-controlplane -n kube-system and look for --authorization-mode
- kubectl get role
  kubectl get rolebindings
  kubectl get clusterrole
  kubectl get clusterrolebindings
  kubectl describe role <>
  kubectl describe rolebinding <>
- To check access
  kubectl auth can-i create deployments
  kubectl auth can-i delete nodes
  kubectl auth can-i create deployments --as <user>
  kubectl auth can-i create pods --as <user> --namespace <>
- To get the list of namespaced and cluster wide resources.
  You can also get resource name (pod), shortname (po), apiversion (v1), kind (Pod), verb (create, delete) and api group (core api depicted as "") from below commands.
  kubectl api-resources --namespaced=true
  kubectl api-resources --namespaced=false
  kubectl api-resources
  kubectl api-resources -o wide

- To limit a role to a specific resource, use resourceNames option.
- example of resources are pods, deployments, nodes, secrets, services, jobs, configmaps, nodes
  example of verbs are get, list, create, update, delete, watch
- A RoleBinding or ClusterRoleBinding binds a role to subjects. Subjects can be groups, users or ServiceAccounts.
  
  
# JSON PATH
- $ for top level hash which has no name aka root level element e.g. $.car
- $ for top level array which has no name aka root level element e.g. $[0]
- . for hash denoted by {}
- [] for array
- All results of JSON Path query are encapsulated with in an array.
- $.*.color -- wild cards are allowed
  $[*].model
- $[0,3] -- first and fourth element
  $[0:3] -- from first upto fourth element, not including fourth element.
  $[-1] or $[-1:] -- last item in list
  $[-4:] -- last 4 items in list
  $[-8:-2] -- from eighth last to third last item in list
- kubectl get po -o json
- kubectl get po -o jsonpath='{JSON PATH Query}'
  kubectl get po orange -o jsonpath='{$.apiVersion}'
  kubectl get po orange -o jsonpath='{$.apiVersion}{"\n"}{$.kind}'
  kubectl get po orange -o jsonpath='{$.apiVersion}{"\t"}{$.kind}'
  kubectl get nodes -o=custom-columns=<COLUMN NAME>:<JSON PATH>
  kubectl get nodes -o=custom-columns=NODE:.metadata.name ,CPU:.status.capacity.cpu
- To find number of nodes which are ready (not including nodes tainted NoSchedule) and write the number to text file output.txt.
  kubectl get nodes -o jsonpath='{range .items[*]}{@.metadata.name}{"\t"}{.spec.taints[*].effect}{"\n"}{end}' | grep -v "NoSchedule" | wc -l 


# yaml files for kube-apiserver, kube-controller-manager, kube-scheduler and etcd are placed in /etc/kubernetes/manifests by default.
# ps -ef | grep kubelet for checking above path.
# For checking the configured options:
  ps -ef | grep kube-scheduler
  ps -ef | grep kube-controller-manager
  ps -ef | grep kube-apiserver
  
# kubelet runs on master node as well as worker nodes.

# Things to remember during the exam:
- Make sure that you apply all the yaml files that you are creating.
- Before dropping / editing any object, take backup of it using yaml file.
- Wherever possible, check your solutions.

# Good Link for network policies:
https://github.com/ahmetb/kubernetes-network-policy-recipes
