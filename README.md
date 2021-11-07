# k3s-hcloud


### Kubernetes stack on Hetzner Cloud (k3s-based)

This [terraform](https://www.terraform.io/) module will install a High Availability [K3s](https://k3s.io/) Cluster with Embedded DB in a private network on [Hetzner Cloud](https://www.hetzner.com/de/cloud). The following resources are provisionised by default (**20€/mo**):

- 3x Control-plane: _CX11_, 2GB RAM, 1VCPU, 20GB NVMe, 20TB Traffic.
- 2x Worker: _CX21_, 4GB RAM, 2VCPU, 40GB NVMe, 20TB Traffic.
- Network: Private network with one subnet.
- Server and agent nodes are distributed across 3 Datacenters (nbg1, fsn1, hel1) for high availability.

</br>
</br>

**Hetzner Cloud integration**:

- Preinstalled [CSI-driver](https://github.com/hetznercloud/csi-driver) for volume support.
- Preinstalled [Cloud Controller Manager for Hetzner Cloud](https://github.com/hetznercloud/hcloud-cloud-controller-manager) for Load Balancer support.

**Auto-K3s-Upgrades**

We provide an example how to upgrade your K3s node and agents with the [system-upgrade-controller](https://github.com/rancher/system-upgrade-controller). Check out [/upgrade](./upgrade)

**What is K3s?**

K3s is a lightweight certified kubernetes distribution. It's packaged as single binary and comes with solid defaults for storage and networking but we replaced [local-path-provisioner](https://github.com/rancher/local-path-provisioner) with hetzner [CSI-driver](https://github.com/hetznercloud/csi-driver) and [klipper load-balancer](https://github.com/k3s-io/klipper-lb) with hetzner [Cloud Controller Manager](https://github.com/hetznercloud/hcloud-cloud-controller-manager). The default ingress controller (traefik) has been disabled.

## Usage

See a more detailed example with walk-through in the [example folder](./example).

<!-- BEGINNING OF PRE-COMMIT-TERRAFORM DOCS HOOK -->
### Inputs

| Name | Description | Type | Default | Required |
|------|-------------|------|---------|:--------:|
| <a name="input_agent_groups"></a> [agent\_groups](#input\_agent\_groups) | Configuration of agent groups | <pre>map(object({<br>    type      = string<br>    count     = number<br>    ip_offset = number<br>  }))</pre> | <pre>{<br>  "default": {<br>    "count": 2,<br>    "ip_offset": 33,<br>    "type": "cx21"<br>  }<br>}</pre> | no |
| <a name="input_control_plane_server_count"></a> [control\_plane\_server\_count](#input\_control\_plane\_server\_count) | Number of control plane nodes | `number` | `3` | no |
| <a name="input_control_plane_server_type"></a> [control\_plane\_server\_type](#input\_control\_plane\_server\_type) | Server type of control plane servers | `string` | `"cx11"` | no |
| <a name="input_create_kubeconfig"></a> [create\_kubeconfig](#input\_create\_kubeconfig) | Create a local kubeconfig file to connect to the cluster | `bool` | `true` | no |
| <a name="input_hcloud_csi_driver_version"></a> [hcloud\_csi\_driver\_version](#input\_hcloud\_csi\_driver\_version) | n/a | `string` | `"v1.5.3"` | no |
| <a name="input_hcloud_token"></a> [hcloud\_token](#input\_hcloud\_token) | Token to authenticate against Hetzner Cloud | `any` | n/a | yes |
| <a name="input_k3s_version"></a> [k3s\_version](#input\_k3s\_version) | K3s version | `string` | `"v1.21.3+k3s1"` | no |
| <a name="input_kubeconfig_filename"></a> [kubeconfig\_filename](#input\_kubeconfig\_filename) | Specify the filename of the created kubeconfig file (defaults to kubeconfig-${var.name}.yaml | `any` | `null` | no |
| <a name="input_name"></a> [name](#input\_name) | Cluster name (used in various places, don't use special chars) | `any` | n/a | yes |
| <a name="input_network_cidr"></a> [network\_cidr](#input\_network\_cidr) | Network in which the cluster will be placed | `string` | `"10.0.0.0/16"` | no |
| <a name="input_server_additional_packages"></a> [server\_additional\_packages](#input\_server\_additional\_packages) | Additional packages which will be installed on node creation | `list(string)` | `[]` | no |
| <a name="input_server_locations"></a> [server\_locations](#input\_server\_locations) | Server locations in which servers will be distributed | `list(string)` | <pre>[<br>  "nbg1",<br>  "fsn1",<br>  "hel1"<br>]</pre> | no |
| <a name="input_ssh_private_key_location"></a> [ssh\_private\_key\_location](#input\_ssh\_private\_key\_location) | Use this private SSH key instead of generating a new one (Attention: Encrypted keys are not supported) | `string` | `null` | no |
| <a name="input_subnet_cidr"></a> [subnet\_cidr](#input\_subnet\_cidr) | Subnet in which all nodes are placed | `string` | `"10.0.1.0/24"` | no |

### Outputs

| Name | Description |
|------|-------------|
| <a name="output_agents_public_ips"></a> [agents\_public\_ips](#output\_agents\_public\_ips) | The public IP addresses of the agent servers |
| <a name="output_control_planes_public_ips"></a> [control\_planes\_public\_ips](#output\_control\_planes\_public\_ips) | The public IP addresses of the control plane servers |
| <a name="output_k3s_token"></a> [k3s\_token](#output\_k3s\_token) | Secret k3s authentication token |
| <a name="output_kubeconfig"></a> [kubeconfig](#output\_kubeconfig) | Structured kubeconfig data to supply to other providers |
| <a name="output_kubeconfig_file"></a> [kubeconfig\_file](#output\_kubeconfig\_file) | Kubeconfig file content with external IP address |
| <a name="output_network_id"></a> [network\_id](#output\_network\_id) | n/a |
| <a name="output_ssh_private_key"></a> [ssh\_private\_key](#output\_ssh\_private\_key) | Key to SSH into nodes |
<!-- END OF PRE-COMMIT-TERRAFORM DOCS HOOK -->

## Common Operations

### Agent server replacement

If you need to cycle an agent, you can do that with a single node following this procedure.
Replace the group name and number with the server you want to recreate!

Make sure you drain the nodes first. 

```shell
terraform taint 'module.my_cluster.module.agent_group["GROUP_NAME"].random_pet.agent_suffix[1]'
terraform apply
```

This will recreate the agent in that group on next apply.

### Control Plane server replacement

Currently you should only replace the servers which didn't initialize the cluster.

```shell
terraform taint 'module.my_cluster.hcloud_server.control_plane["#1"]'
terraform apply
```

## Auto-Upgrade

### Prerequisite

Install the system-upgrade-controller in your cluster.

```
KUBECONFIG=kubeconfig.yaml kubectl apply -f ./upgrade/controller.yaml
```

## Upgrade procedure

1. Mark the nodes you want to upgrade (The script will mark all nodes).

```
KUBECONFIG=kubeconfig.yaml kubectl label --all node k3s-upgrade=true
```

2. Run the plan for the **servers**.

```
KUBECONFIG=kubeconfig.yaml kubectl apply -f ./upgrade/server-plan.yaml
```

**Warning:** Wait for completion [before you start upgrading your agents](https://github.com/k3s-io/k3s/issues/2996#issuecomment-788352375).

3. Run the plan for the **agents**.

```
KUBECONFIG=kubeconfig.yaml kubectl apply -f ./upgrade/agent-plan.yaml
```

## Backups

K3s will automatically backup your embedded etcd datastore every 12 hours to `/var/lib/rancher/k3s/server/db/snapshots/`.
You can reset the cluster by pointing to a specific snapshot.

1. Stop the master server.

```sh
sudo systemctl stop k3s
```

2. Restore the master server with a snapshot

```sh
./k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=<PATH-TO-SNAPSHOT>
```

**Warning:** This forget all peers and the server becomes the sole member of a new cluster. You have to manually rejoin all servers.

3. Connect you with the different servers backup and delete `/var/lib/rancher/k3s/server/db` on each peer etcd server and rejoin the nodes.

```sh
sudo systemctl stop k3s
rm -rf /var/lib/rancher/k3s/server/db
sudo systemctl start k3s
```

This will rejoin the server with the master server and seed the etcd store.

**Info:** It exists no official tool to automate the procedure. In future, rancher might provide an operator to handle this ([issue](https://github.com/k3s-io/k3s/issues/3174)).

## Debugging

Cloud init logs can be found on the remote machines in:

- `/var/log/cloud-init-output.log`
- `/var/log/cloud-init.log`
- `journalctl -u k3s.service -e` last logs of the server
- `journalctl -u k3s-agent.service -e` last logs of the agent

## Known issues

- Sometimes at cluster bootstrapping the Cloud-Controllers reports that some routes couldn't be created. This issue was fixed in master but wasn't released yet. Restart the cloud-controller pod and it will recreate them.

## Credits

- [terraform-hcloud-k3s](https://github.com/cicdteam/terraform-hcloud-k3s) Terraform module which creates a single node cluster.
- [terraform-module-k3](https://github.com/xunleii/terraform-module-k3s) Terraform module which creates a k3s cluster, with multi-server and management features.
- Icon created by [Freepik](https://www.freepik.com) from [www.flaticon.com](https://www.flaticon.com/de/)

## Example:

```
$ terraform apply

var.hcloud_token
  Token to authenticate against Hetzner Cloud

  Enter a value: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

var.name
  Cluster name (used in various places, don't use special chars)

  Enter a value: k3s
  
  ...
  
  
  Enter a value: yes

...

Apply complete! Resources: 17 added, 0 changed, 0 destroyed.

Outputs:

agents_public_ips = [
  "49.12.225.212",
  "23.88.123.16",
]
control_planes_public_ips = [
  "23.88.107.54",
  "49.12.5.242",
  "65.108.52.118",
]
k3s_token = <sensitive>
kubeconfig = <sensitive>
kubeconfig_file = <sensitive>
network_id = "1260727"
ssh_private_key = <sensitive>
```

Check k8s:
```
$ export KUBECONFIG=./kubeconfig-k3s.yaml

$ kubectl get node -o wide
NAME                          STATUS   ROLES                       AGE    VERSION        INTERNAL-IP   EXTERNAL-IP     OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
k3s-control-plane-0           Ready    control-plane,etcd,master   3m     v1.21.3+k3s1   10.0.1.1      23.88.107.54    Ubuntu 20.04.3 LTS   5.4.0-89-generic   containerd://1.4.8-k3s1
k3s-control-plane-1           Ready    control-plane,etcd,master   71s    v1.21.3+k3s1   10.0.1.2      49.12.5.242     Ubuntu 20.04.3 LTS   5.4.0-89-generic   containerd://1.4.8-k3s1
k3s-control-plane-2           Ready    control-plane,etcd,master   88s    v1.21.3+k3s1   10.0.1.3      65.108.52.118   Ubuntu 20.04.3 LTS   5.4.0-89-generic   containerd://1.4.8-k3s1
k3s-default-0-expert-marten   Ready    <none>                      101s   v1.21.3+k3s1   10.0.1.33     49.12.225.212   Ubuntu 20.04.3 LTS   5.4.0-89-generic   containerd://1.4.8-k3s1
k3s-default-1-suited-hawk     Ready    <none>                      109s   v1.21.3+k3s1   10.0.1.34     23.88.123.16    Ubuntu 20.04.3 LTS   5.4.0-89-generic   containerd://1.4.8-k3s1

$ kubectl get all --all-namespaces
NAMESPACE     NAME                                                   READY   STATUS    RESTARTS   AGE
kube-system   pod/coredns-7448499f4d-w9ztt                           1/1     Running   0          2m53s
kube-system   pod/hcloud-cloud-controller-manager-74b74b9b46-hk4wq   1/1     Running   0          2m48s
kube-system   pod/hcloud-csi-controller-0                            5/5     Running   0          2m47s
kube-system   pod/hcloud-csi-node-b6fdc                              3/3     Running   0          2m46s
kube-system   pod/hcloud-csi-node-mlhnx                              3/3     Running   0          92s
kube-system   pod/hcloud-csi-node-qzp2t                              3/3     Running   0          2m9s
kube-system   pod/hcloud-csi-node-sgwrw                              3/3     Running   0          109s
kube-system   pod/hcloud-csi-node-w9556                              3/3     Running   0          2m2s
kube-system   pod/metrics-server-86cbb8457f-sw4rm                    1/1     Running   0          2m53s

NAMESPACE     NAME                                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default       service/kubernetes                      ClusterIP   10.43.0.1       <none>        443/TCP                  3m13s
kube-system   service/hcloud-csi-controller-metrics   ClusterIP   10.43.239.98    <none>        9189/TCP                 2m47s
kube-system   service/hcloud-csi-node-metrics         ClusterIP   10.43.251.254   <none>        9189/TCP                 2m46s
kube-system   service/kube-dns                        ClusterIP   10.43.0.10      <none>        53/UDP,53/TCP,9153/TCP   3m10s
kube-system   service/metrics-server                  ClusterIP   10.43.228.154   <none>        443/TCP                  3m8s

NAMESPACE     NAME                             DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
kube-system   daemonset.apps/hcloud-csi-node   5         5         5       5            5           <none>          2m47s

NAMESPACE     NAME                                              READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/coredns                           1/1     1            1           3m11s
kube-system   deployment.apps/hcloud-cloud-controller-manager   1/1     1            1           2m49s
kube-system   deployment.apps/metrics-server                    1/1     1            1           3m10s

NAMESPACE     NAME                                                         DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/coredns-7448499f4d                           1         1         1       2m55s
kube-system   replicaset.apps/hcloud-cloud-controller-manager-74b74b9b46   1         1         1       2m49s
kube-system   replicaset.apps/metrics-server-86cbb8457f                    1         1         1       2m55s

NAMESPACE     NAME                                     READY   AGE
kube-system   statefulset.apps/hcloud-csi-controller   1/1     2m48s

```

<img src="pictures/k3s-hcloud-servers.png" width="900">
<img src="pictures/k3s-hcloud-networks.png" width="900">
<img src="pictures/k3s-hcloud-ssh_keys.png" width="900">


Check LB $ PVC :
```
$ kubectl apply -f manifests/hello-kubernetes-default.yaml 
deployment.apps/hello-kubernetes created
service/hello-kubernetes created
persistentvolumeclaim/csi-pvc created


$ kubectl get all 
NAME                                    READY   STATUS    RESTARTS   AGE
pod/hello-kubernetes-6f8d7694bc-xstlz   1/1     Running   0          2m38s

NAME                       TYPE           CLUSTER-IP      EXTERNAL-IP                                    PORT(S)          AGE
service/hello-kubernetes   LoadBalancer   10.43.231.169   10.0.1.4,162.55.152.168,2a01:4f8:1c1d:178::1   8080:30797/TCP   2m38s
service/kubernetes         ClusterIP      10.43.0.1       <none>                                         443/TCP          9m34s

NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-kubernetes   1/1     1            1           2m38s

NAME                                          DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-kubernetes-6f8d7694bc   1         1         1       2m38s

```

<img src="pictures/k3s-hcloud-hello-volume.png" width="900">
<img src="pictures/k3s-hcloud-hello-load_balancer.png" width="900">
<img src="pictures/k3s-hcloud-hello.png" width="900">

Clean: 

delete LB via hcloud UI and -> 

$ terraform destroy:


```
var.hcloud_token
  Token to authenticate against Hetzner Cloud

  Enter a value: XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX

var.name
  Cluster name (used in various places, don't use special chars)

  Enter a value: k3s

...

  Enter a value: yes



Destroy complete! Resources: 17 destroyed.


```


