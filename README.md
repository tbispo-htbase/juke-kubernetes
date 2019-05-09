Deploy a Juke Kubernetes Cluster
============================================

Quick Start
-----------

#### Usage

    # Install dependencies from ``requirements.txt``
    sudo pip install -r requirements.txt

    # Copy ``inventory/sample`` as ``inventory/mycluster``
    cp -rfp inventory/sample inventory/mycluster

    # Update Ansible inventory file with inventory builder
    declare -a IPS=(10.10.1.3 10.10.1.4 10.10.1.5)
    CONFIG_FILE=inventory/mycluster/hosts.yml python3 contrib/inventory_builder/inventory.py ${IPS[@]}

    # Review and change parameters under ``inventory/mycluster/group_vars``
    cat inventory/mycluster/group_vars/all/all.yml
    cat inventory/mycluster/group_vars/k8s-cluster/k8s-cluster.yml

    # Deploy Kubespray with Ansible Playbook - run the playbook as root
    # The option `-b` is required, as for example writing SSL keys in /etc/,
    # installing packages and interacting with various systemd daemons.
    # Without -b the playbook will fail to run!
    ansible-playbook -i inventory/mycluster/hosts.yml --become --become-user=root cluster.yml


Adding nodes
------------

You may want to add worker, master or etcd nodes to your existing cluster. This can be done by re-running the `cluster.yml` playbook, or you can target the bare minimum needed to get kubelet installed on the worker and talking to your masters. This is especially helpful when doing something like autoscaling your clusters.

-   Add the new worker node to your inventory in the appropriate group.
-   Run the ansible-playbook command, substituting `cluster.yml` for `scale.yml`:

        ansible-playbook -i inventory/mycluster/hosts.ini scale.yml -b -v \
          --private-key=~/.ssh/private_key

Removing nodes
------------

You may want to remove **worker** nodes to your existing cluster. This can be done by re-running the `remove-node.yml` playbook. First, all nodes will be drained, then stop some kubernetes services and delete some certificates, and finally execute the kubectl command to delete these nodes. This can be combined with the add node function, This is generally helpful when doing something like autoscaling your clusters. Of course if a node is not working, you can remove the node and install it again.

Add worker nodes to the list under kube-node if you want to delete them.

    ansible-playbook -i inventory/mycluster/hosts.ini remove-node.yml -b -v \
        --private-key=~/.ssh/private_key

Use `--extra-vars "node=<nodename>,<nodename2>"` to select the node you want to delete.
```
ansible-playbook -i inventory/mycluster/hosts.ini remove-node.yml -b -v \
  --private-key=~/.ssh/private_key \
  --extra-vars "node=nodename,nodename2"
```

Deploying Dashboard
------------

In order to deploy the Kubernetes dashboard, the following command has to be executed inside the Kubernetes Master node:

Deploying the Kubernetes dashboard:

    kubectl apply -f https://raw.githubusercontent.com/htbase/juke-kubernetes/master/kubernetes-dashboard.yaml

Deploy required access controls to dashboard:
```
kubectl apply -f https://raw.githubusercontent.com/htbase/juke-kubernetes/master/kube-dashboard-access.yaml"


Supported Linux Distributions
-----------------------------

-   **Container Linux by CoreOS**
-   **Debian** Buster, Jessie, Stretch, Wheezy
-   **Ubuntu** 16.04, 18.04
-   **CentOS/RHEL** 7
-   **Fedora** 28
-   **Fedora/CentOS** Atomic
-   **openSUSE** Leap 42.3/Tumbleweed

Note: Upstart/SysV init based OS types are not supported.

Supported Components
--------------------

-   Core
    -   [kubernetes](https://github.com/kubernetes/kubernetes) v1.13.5
    -   [etcd](https://github.com/coreos/etcd) v3.2.26
    -   [docker](https://www.docker.com/) v18.06 (see note)
    -   [rkt](https://github.com/rkt/rkt) v1.21.0 (see Note 2)
    -   [cri-o](http://cri-o.io/) v1.11.5 (experimental: see [CRI-O Note](docs/cri-o.md). Only on centos based OS)
-   Network Plugin
    -   [calico](https://github.com/projectcalico/calico) v3.4.0
    -   [canal](https://github.com/projectcalico/canal) (given calico/flannel versions)
    -   [cilium](https://github.com/cilium/cilium) v1.3.0
    -   [contiv](https://github.com/contiv/install) v1.2.1
    -   [flanneld](https://github.com/coreos/flannel) v0.11.0
    -   [kube-router](https://github.com/cloudnativelabs/kube-router) v0.2.5
    -   [multus](https://github.com/intel/multus-cni) v3.1.autoconf
    -   [weave](https://github.com/weaveworks/weave) v2.5.1
-   Application
    -   [cert-manager](https://github.com/jetstack/cert-manager) v0.5.2
    -   [coredns](https://github.com/coredns/coredns) v1.4.0
    -   [ingress-nginx](https://github.com/kubernetes/ingress-nginx) v0.21.0

Note: The list of validated [docker versions](https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG-1.13.md) was updated to 1.11.1, 1.12.1, 1.13.1, 17.03, 17.06, 17.09, 18.06. kubeadm now properly recognizes Docker 18.09.0 and newer, but still treats 18.06 as the default supported version. The kubelet might break on docker's non-standard version numbering (it no longer uses semantic versioning). To ensure auto-updates don't break your cluster look into e.g. yum versionlock plugin or apt pin).

Requirements
------------

-   **Ansible v2.7.6 (or newer) and python-netaddr is installed on the machine
    that will run Ansible commands**
-   **Jinja 2.9 (or newer) is required to run the Ansible Playbooks**
-   The target servers must have **access to the Internet** in order to pull docker images. Otherwise, additional configuration is required (See [Offline Environment](https://github.com/kubernetes-sigs/kubespray/blob/master/docs/downloads.md#offline-environment))
-   The target servers are configured to allow **IPv4 forwarding**.
-   **Your ssh key must be copied** to all the servers part of your inventory.
-   The **firewalls are not managed**, you'll need to implement your own rules the way you used to.
    in order to avoid any issue during deployment you should disable your firewall.
-   If kubespray is ran from non-root user account, correct privilege escalation method
    should be configured in the target servers. Then the `ansible_become` flag
    or command parameters `--become or -b` should be specified.

Hardware:        
These limits are safe guarded by Juke. Actual requirements for your workload can differ. For a sizing guide go to the [Building Large Clusters](https://kubernetes.io/docs/setup/cluster-large/#size-of-master-and-master-components) guide.

-   Master
    - Memory: 1500 MB
-   Node
    - Memory: 1024 MB

Network Plugins
---------------

You can choose between 6 network plugins. (default: `calico`, except Vagrant uses `flannel`)

-   [flannel](docs/flannel.md): gre/vxlan (layer 2) networking.

-   [calico](docs/calico.md): bgp (layer 3) networking.

-   [canal](https://github.com/projectcalico/canal): a composition of calico and flannel plugins.

-   [cilium](http://docs.cilium.io/en/latest/): layer 3/4 networking (as well as layer 7 to protect and secure application protocols), supports dynamic insertion of BPF bytecode into the Linux kernel to implement security services, networking and visibility logic.

-   [contiv](docs/contiv.md): supports vlan, vxlan, bgp and Cisco SDN networking. This plugin is able to
    apply firewall policies, segregate containers in multiple network and bridging pods onto physical networks.

-   [weave](docs/weave.md): Weave is a lightweight container overlay network that doesn't require an external K/V database cluster.
    (Please refer to `weave` [troubleshooting documentation](http://docs.weave.works/weave/latest_release/troubleshooting.html)).

-   [kube-router](docs/kube-router.md): Kube-router is a L3 CNI for Kubernetes networking aiming to provide operational
    simplicity and high performance: it uses IPVS to provide Kube Services Proxy (if setup to replace kube-proxy),
    iptables for network policies, and BGP for ods L3 networking (with optionally BGP peering with out-of-cluster BGP peers).
    It can also optionally advertise routes to Kubernetes cluster Pods CIDRs, ClusterIPs, ExternalIPs and LoadBalancerIPs.

-   [multus](docs/multus.md): Multus is a meta CNI plugin that provides multiple network interface support to pods. For each interface Multus delegates CNI calls to secondary CNI plugins such as Calico, macvlan, etc.

The choice is defined with the variable `kube_network_plugin`. There is also an
option to leverage built-in cloud provider networking instead.
See also [Network checker](docs/netcheck.md).
