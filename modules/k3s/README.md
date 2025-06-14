# k3s module

Module for provisioning k3s clusters in proxmox. Currently this module relies on having an internet connection to grab the k3s install script and to pull k3s dependencies. In the future it may be updated to support disconnected installations as well when paired with images built for disconnected installs.

Uses bpg proxmox provider.

# Assumptions
- Currently expects to clone a Debian/Ubuntu proxmox template that has the qemu-guest-agent already installed. Tested with template built on Ubuntu server minimal cloud image version 22.04 with qemu-guest-agent preconfigured
- Relies on having SSH access to proxmox nodes in order to create snippets used for configuring cloud-init userdata and metadata
- Expects the same template to be available on each Proxmox node for VM cloning

# Configuration notes

## Node count and location
Since Proxmox requires choosing a specific node to create a VM on, this module relies on passing lists of nodes to control the count and location of k3s nodes. For example setting `proxmox_server_nodes = ["pve0", "pve1", "pve2"]` would create 1 k3s server node VM on each of the listed proxmox hosts. Setting `proxmox_server_nodes = ["pve0", "pve0", "pve0"]` would instead create 3 k3s server node VMs all on the proxmox host named `pve0`. It is a little clunky compared to just passing in a count, but it is a proxmox-ism.

Note that [you should always run an odd number of server nodes](https://docs.k3s.io/datastore/ha-embedded) to prevent ties when determining a quorum since this module currently uses embedded etcd.

### Agent nodes
By default all k3s server nodes are also agent nodes and don't have any taints applied to them so any workload can be scheduled on them. If you wish to deploy dedicated agent nodes which don't run the control-plane or etcd components, you can configure the `proxmox_agent_nodes` input variable the same way you do the `proxmox_server_nodes`. The value of these 2 input vars don't need to be the same and you can deploy server and agent nodes to completely different hosts or they can share proxmox hosts.

examples: Setting `proxmox_server_nodes = ["pve0", "pve1", "pve2"]` and `proxmox_agent_nodes = ["pve0", "pve1", "pve2"]` would deploy 1 server and 1 agent node to each of the 3 listed proxmox hosts. Setting `proxmox_server_nodes = ["pve0", "pve1", "pve2"]` and `proxmox_agent_nodes = ["pve3", "pve4"]` would deploy 3 server nodes and 2 agent nodes, and each of them would be running on a unique host.

## kube API access
The [k3s-basic-deployment](../../examples/k3s-basic-deployment/) example of using this module shows setting the `server_hostname` input variable to the static IP passed to the bootstrap node. This is fine for an example and testing purposes, but it would be more resilient and flexible to set this to a hostname or IP of an external load balancer instead. The hostname or IP should be configured to route to a load balancer service, such as haproxy or nginx, running outside of the cluster that balances traffic for port 6443 between each server node. This enables new cluster nodes to join the cluster by hitting any of the ready server nodes, not just the bootstrap node. It also enables more resilient connections to the kube api for interacting with your cluster since you aren't configuring your kube context based on an IP of a single node.

The [k3s-with-load-balancer-deployment](../../examples/k3s-with-load-balancer-deployment) example can be used as a reference to configure this k3s module alongside the [haproxy-lb module](../haproxy-lb/) to deploy a high availability TCP load balancer for the kubernetes API. This load balancer is then used by nodes when joining the cluster as well as by tools like kubectl when interacting with the kubernetes API.

# How to access your cluster after it is deployed
Once it is deployed you can connect to any of the server nodes to get the kubeconfig. Would recommend using the bootstrap node (the vm that includes `server-0` in the name) initially to confirm that your cluster nodes are actually joining since that node is the one that starts the cluster.

Once you are on one of your server nodes, run `cloud-init status --wait` to ensure that cloud-init has finished running and completed successfully. After it has, you can access the kubeconfig for the cluster located at `/etc/rancher/k3s/k3s.yaml`. This file is owned by root so to use it from the server node you are connected to as your SSH user you can run `chown $USER:$USER /etc/rancher/k3s/k3s.yaml`. After doing that you can run `kubectl get nodes` and watch for all of your server and agent nodes (if deployed) to join the cluster.

kubectl installed by the k3s installer automatically loads the kubeconfig from that location, so while on your server nodes you just need to have permission to that file. To connect to the cluster from a different machine such as your dev computer or a different Proxmox vm, copy the contents of `/etc/rancher/k3s/k3s.yaml` and paste it into a file (probably just `~/.ssh/config` if you don't already have a preferred method of managing kube contexts) on the machine you wish to access the cluster from. Make sure to edit the value of the server hostname from `server: https://127.0.0.1:6443` to replace `127.0.0.1` with whatever you set the `server_hostname` input variable to for the k3s module. This will likely be your bootstrap node IP address if you are just copying the example, but it could also be a hostname of load balancer if you have one configured for your kube api.

There are many ways to manage and change your kube context. The simplest method is to paste the kubeconfig contents to `~/.kube/config` and tools like kubectl will automatically use it by default. It can also be placed somewhere else and the env var `KUBECONFIG` can be set to the path to the file to change your context. I personally like to keep a separate kubeconfig file per cluster in my `~/.kube` directory and use a tool like [kubie](https://github.com/sbstp/kubie) to manage my context. If you don't already have a preferred method, I would recommend trying a context manager like kubie out.

# Known issues and limitations
- Currently only deploys with a root disk and doesn't configure additional data volumes
  - Currently it uses the same virtual disk settings for the root volume for both server and agent nodes
- ~~Currently only deploys server nodes, no dedicated agent nodes~~ This has been added as of version 0.1.3
- This module can be used to bootstrap new clusters and join nodes to existing clusters, but it does not automatically handle nodes being removed from the cluster or cluster upgrades. Without using other tooling for cluster upgrades, the upgrade path using only this module would be to deploy a new set of upgraded nodes, join them to your cluster, and then manually cordon and drain the old nodes. Then once workloads have migrated to the new nodes, you could destroy the old node VMs with a tofu/terraform destroy
- If the template being cloned has any settings configured that the VM definitions in this module don't configure, an initial deploy will work fine but subsequent deploys will want to revert those settings from the template to match the defaults that the provider uses. This will appear as terraform/tofu wanting to change settings to null or default values if the provider has any.
  - If there are any default cloud-init settings on the template for configuring a user, password, or ssh key then a subsequent apply will want to destroy and recreate the VMs because cloud-init changes force a recreation. It is recommended to remove these settings from the templates being used with this module
