# enable-swap-k3s-notes

These instructions help on enabling swap on K8 / k3s. Please notice that with this setup, you embrace LimitedSwap approach (swap availability for each burstable container is a ratio between its requested memory and the total memory of the system, multiplied by the total system swap size). Alternatively, you could use UnlimitedSwap where all containers can use the available swap space.

- First, create a file /etc/rancher/k3s/kubeletconfig.yaml with the next content
```
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
memorySwap:
  swapBehavior: LimitedSwap
```
- Then, edit content of /etc/rancher/k3s/config.yaml (create the file if it doesn't exists):
```
kubelet-arg:
   - "config=/etc/rancher/k3s/kubeletconfig.yaml"
kube-apiserver-arg:
   - "feature-gates=NodeSwap=true"
```
- Then reload systemcl config
```
systemctl daemon-reload
```
- And restart k3s
```
systemctl restart k3s
```
- Now, check for 1 at the end of the response to this command, which would shows that NodeSwap is enabled
```
kubectl get --raw /metrics  | grep kubernetes_feature_enabled | grep Swap
```
- launch kubectl proxy
```
kubectl proxy
```
- And in another terminal:
```
curl -X GET http://127.0.0.1:8001/api/v1/nodes/<node-name>/proxy/configz | jq . | grep Swap
```
- You should see:
```
"memorySwap": {
      "swapBehavior": "LimitedSwap"
```
- To test this is working, create a container with the specs:
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
  - name: c1
    resources:
      requests:
        memory: "512M"
    image: quay.io/mabekitzur/fedora-with-stress-ng:12-3-2022
    command: ["/bin/bash"]
    args: ["-c", "sleep 9999"]
```
- Then, check swap limit:
```
kubectl exec -it test-pod -c c1 -- "cat" "/sys/fs/cgroup/memory.swap.max"
182378123
```
- Then try to stress it
- In one command line run
```
kubectl exec -it test-pod -c c1 -- batch
--> while true; do cat /sys/fs/cgroup/memory.swap.current; sleep 2; done
```
- In another command line run:
```
kubectl exec -it test-pod -c c1 -- batch
--> stress-ng --vm 2 --vm-bytes 5G --mmap 2 --mmap-bytes 5G --vm-keep --timeout 2m -m 1 --metrics
```
- The first command line screen should start going up from 0 to the memory.swap.max value
```
0
....
432134
2134211
...
```

# Notes:
- Remember, swap will only work in Burstable QoS Pods/Containers
- swap usage may depend on sysctl vm.swappiness (change it with sysctl vm.swappiness=xx)
- you should first have created a swap file and enable it (or having a swap partition):
  - ```
    sudo fallocate -l 1G /swapfile
    sudo chmod 600 /swapfile
    sudo mkswap /swapfile
    sudo swapon /swapfile
    sudo echo "/swapfile swap swap defaults 0 0" > /etc/fstab
    ```
  - Check it is working with
    ```
    sudo swapon --show
    ```
  - Negative values in affinity columns are ok (default for ubuntu managed swap)
- Outside of k3s, arguments are given directly to kubelet or kubeadm on cluster creation. Also for other kubernetes version, you may need to disable the --fail-swap-on flag for the cluster to start when swap is available
