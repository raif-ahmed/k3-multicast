# Configure multicast on K3s 

## References 
- https://www.spectric.com/post/multicast-within-kubernetes
- [Deterministic cni-bin-dir](https://github.com/k3s-io/k3s/issues/1434)
  
## Deploy multus to K3s

The first step is to deploy multus to k3s, so we will need to inspect the correct paths for cni & cnibin



```bash
journalctl -u k3s|grep cni-conf-dir
Sep 29 16:05:44 tst-st-srv1 k3s[28281]: time="2022-09-29T16:05:44.097107228Z" level=info msg="Running kubelet --address=0.0.0.0 --anonymous-auth=false --authentication-token-webhook=true --authorization-mode=Webhook --cgroup-driver=cgroupfs --client-ca-file=/var/lib/rancher/k3s/agent/client-ca.crt --cloud-provider=external --cluster-dns=10.43.0.10 --cluster-domain=cluster.local --cni-bin-dir=/var/lib/rancher/k3s/data/9de9bfcf367b723ef0ac73dd91761165a4a8ad11ad16a758d3a996264e60c612/bin --cni-conf-dir=/var/lib/rancher/k3s/agent/etc/cni/net.d --container-runtime-endpoint=unix:///run/k3s/containerd/containerd.sock --container-runtime=remote --containerd=/run/k3s/containerd/containerd.sock --eviction-hard=imagefs.available<5%,nodefs.available<5% --eviction-minimum-reclaim=imagefs.available=10%,nodefs.available=10% --fail-swap-on=false --healthz-bind-address=127.0.0.1 --hostname-override=tst-st-srv1 --kubeconfig=/var/lib/rancher/k3s/agent/kubelet.kubeconfig --node-labels= --pod-manifest-path=/var/lib/rancher/k3s/agent/pod-manifests --read-only-port=0 --resolv-conf=/etc/resolv.conf --serialize-image-pulls=false --tls-cert-file=/var/lib/rancher/k3s/agent/serving-kubelet.crt --tls-private-key-file=/var/lib/rancher/k3s/agent/serving-kubelet.key"
Sep 29 16:05:44 tst-st-srv1 k3s[28281]: Flag --cni-conf-dir has been deprecated, will be removed along with dockershim.
```
> So we have 
> * --cni-bin-dir=/var/lib/rancher/k3s/data/9de9bfcf367b723ef0ac73dd91761165a4a8ad11ad16a758d3a996264e60c612/bin 
> * --cni-conf-dir=/var/lib/rancher/k3s/agent/etc/cni/net.d

So we need to change those two parameters on multus daemonset.

To do this I will patch https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml using [kustomization](kustomization.yaml).

```yaml
- target:
    group: apps
    kind: DaemonSet
    name: kube-multus-ds
    namespace: kube-system
  patch: |-
    - op: add
      path: /spec/template/spec/containers/0/args/2
      value: "--multus-kubeconfig-file-host=/var/lib/rancher/k3s/agent/etc/cni/net.d/multus.d/multus.kubeconfig"
    - op: replace
      path: /spec/template/spec/volumes/0/hostPath/path
      value: "/var/lib/rancher/k3s/agent/etc/cni/net.d"
    - op: replace
      path: /spec/template/spec/volumes/1/hostPath/path
      value: "/var/lib/rancher/k3s/data/9de9bfcf367b723ef0ac73dd91761165a4a8ad11ad16a758d3a996264e60c612/bin"
```

## Configure a macvlan

```yaml
apiVersion: "k8s.cni.cncf.io/v1" 
kind: NetworkAttachmentDefinition 
metadata: 
  name: eth1-multicast 
spec: 
  config: '{ 
      "cniVersion": "0.3.0", 
      "type": "macvlan", 
      "master": "eth1", 
      "mode": "bridge", 
      "ipam": { 
        "type": "host-local", 
        "subnet": "10.0.0.0/24", 
        "rangeStart": "10.0.0.13", 
        "rangeEnd": "10.0.0.254" 
      } 
    }' 
```
Config: { "cniVersion": "0.3.1", "plugins": [ {"name": "oampub", "type": "macvlan", "master": "vlan.1010", "ipam": { "type": "static", "routes": [ { "dst": "10.108.180.100/32", "gw": "10.112.136.209" }, { "dst": "10.108.180.101/32", "gw": "10.112.136.209" }, { "dst": "10.108.180.164/32", "gw": "10.112.136.209" } ] } }, { "type": "sbr" } ] }