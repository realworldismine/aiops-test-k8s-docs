# aiops-test-k8s-docs

## Intial Setting
### Worker Node Connection
- Add Kubernetes and containerd settings and restart
  - Ensure that the filesystem is overlay2
  - Set swap off
```SHELL
swapoff -a
sed -i '/swap/s/^/#/' /etc/fstab
```
  - Save the default containerd settings to config.toml, then change the following value
```TOML
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
  SystemdCgroup = true
```
  - Set sysctl and install modules
```SHELL
root@dev-ubuntu:~# vi /etc/modules-load.d/k8s.conf
br_netfilter

root@dev-ubuntu:~# vi /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-arptables = 1
net.ipv4.ip_forward = 1

root@dev-ubuntu:~# sysctl --system

root@dev-ubuntu:~# apt-get install -y apt-transport-https ca-certificates curl

root@dev-ubuntu:~# curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes.gpg
root@dev-ubuntu:~# echo "deb [signed-by=/etc/apt/keyrings/kubernetes.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

root@dev-ubuntu:~# apt-get update
root@dev-ubuntu:~# apt-get install -y kubelet kubeadm kubectl
root@dev-ubuntu:~# apt-mark hold kubelet kubeadm kubectl
```
- Issue a kubeadm token and join

### Deployment Notes
- Never deploy workloads on the control plane
- Always double-check kube-proxy
- Roll out coredns and restart
```SHELL
kubectl -n kube-system rollout restart deployment coredns
```

### Docker Container Creation and Deployment
- It was more challenging than expected
- Run `apt-get update` before installing required packages
- Create a Python virtual environment and upgrade `pip`
- Install `wheel` first
- Timezone is set to UTC+0, so make sure to apply local time
- Deploy to Docker Hub after building
- Do not use Windows Docker, even if the hardware requirements are met, the system may shut down unexpectedly

### Docker Execution
- Specify the configuration file to use during execution
- Map main directories for external visibility
- The external database address is recognized correctly inside Docker settings
- However, if the Docker image is used in Kubernetes, the external database address is not recognized in the local network, so address mapping is required

### Kubernetes Environment Setup
- Start by creating namespaces
- To monitor workloads externally, create storage class, persistent volume, and persistent volume claim
- To map the external database address and port, create a service and endpoint, then enter the service name in the host name of the config map
  - configmap setting
```YAML
db:
  default: pgsql
  pgsql_option:
    db_host: aiops-db-connection-svc
```
  - Service and Endpoint setting
```YAML
apiVersion: v1
kind: Service
metadata:
  name: aiops-db-connection-svc
spec:
  type: ClusterIP
  ports:
  - name: dockerport
    port: 15432
    targetPort: 15432

---
apiVersion: v1
kind: Endpoints
metadata:
  name: aiops-db-connection-svc
subsets:
- addresses:
  - ip: 192.168.0.42
  ports:
  - name: dockerport
    port: 15432
    protocol: TCP
```
- When creating a persistent volume, set node affinity and specify a storage location for external access
```YAML
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  storageClassName: collector-storage
  local:
    path: /aiops-container/6/data
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - dev-ubuntu
```
- If the external mapping path is different, create a separate persistent volume
```YAML
spec:
  containers:
  - name: collector-container
    image: onikaze/acollector:main
    volumeMounts:
    - name: collector-volume
      mountPath: /app/data
    - name: collector-main-volume
      mountPath: /app/archive
      subPath: archive
...
  volumeClaimTemplates:
  - metadata:
      name: collector-volume
    spec:
      accessModes: ["ReadWriteMany"]
      resources:
        requests:
          storage: 10Gi
      storageClassName: collector-storage
  - metadata:
      name: collector-main-volume
    spec:
      accessModes: ["ReadWriteMany"]
      resources:
        requests:
          storage: 10Gi
      storageClassName: collector-main-storage
```

### Workload Configuration
- Always set the container image to be pulled (depends on the program)
- It is a principle to always set request and limit for each container, but these are temporarily not set due to memory errors on insufficient PC resources
- When mounting a volume, specify each persistent volume, and set subPath for subdirectories
- Config maps can also be mapped when mounting volumes, set the config map variable name as subPath and map the actual path
- Specify the data to apply in the config map
- Enter the persistent volume or config map when specifying volumes
- Assign nodes if needed

### Deployment & StatefulSet
- The main AIOps application is configured with Deployment, and the AIOps collector application is configured with StatefulSet
- It is okay to configure both as either Deployment or StatefulSet
- However, keep in mind that there are differences when configuring persistent volumes
- When configuring a Deployment, all storage class, persistent volume, and persistent volume claim must be pre-created and the persistent volume claim must be mapped
- On the other hand, when configuring a StatefulSet, it is recommended to create the storage class and persistent volume first, then map the storage class, in which case the persistent volume claim is automatically created accordingly
  - Deployment
```YAML
volumes:
  - name: aiops-storage6
    persistentVolumeClaim:
      claimName: aiops-pvc6
  - name: aiops-config6
    configMap:
      name: aiops-cm-6
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - dev-ubuntu
```
  - StatefulSet
```YAML
volumeClaimTemplates:
- metadata:
    name: collector-volume
  spec:
    accessModes: ["ReadWriteMany"]
    resources:
      requests:
        storage: 10Gi
    storageClassName: collector-storage
- metadata:
    name: collector-main-volume
  spec:
    accessModes: ["ReadWriteMany"]
    resources:
      requests:
        storage: 10Gi
    storageClassName: collector-main-storage
```
