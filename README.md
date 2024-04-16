
# Guide to Installing a k8s Cluster on 2 Master Nodes + HAproxy on Proxmox VM Server

## Introduction

![enter image description here](k8s-home.png)

This guide was written to gain experience working with Kubernetes and Ansible. Most of the instructions found were either outdated or contained errors.

## Virtual Machine Information

-   192.168.1.233 - proxy
-   192.168.1.230 - k8s-m1
-   192.168.1.231 - k8s-m2

## Preparation of Virtual Machine Template

Virtual machines are installed by cloning from a prepared template. You need to prepare a virtual machine in advance and convert it into a template. You will find detailed instructions on creating a virtual machine template [here](https://pve.proxmox.com/wiki/VM_Templates_and_Clones).

## Virtual Machine Installation

To run the playbook on a server with Proxmox VM installed, you need to install [proxmoxer](https://pypi.org/project/proxmoxer/).

To install, execute the command:


    apt install python3-proxmoxer

## Inventory File Structure

[pvenodes] - IP proxmox server  
[k8s_master] - IP master VM  
[k8s_master2] - IP master VM 2  
[proxy] - IP HAproxy VM

## vm_vars.yml File Structure

-   proxmox_node - Proxmox node name
-   proxmox_user - root user of the Proxmox node - example root@pam
-   proxmox_password - root password of the Proxmox node
-   proxmox_bridge - name of the bridge interface - example vmbr0
-   proxmox_host - IP address of the Proxmox node
-   template_id - template ID for cloning virtual machines
-   cloudinit_user - username to be created on the virtual machine
-   cloudinit_password - password to be created for the user
-   sshkey - SSH key to be added during virtual machine cloning
-   vms - contains IDs, names, and IPs for creating virtual servers

## Creating Virtual Machines

To install virtual machines, run the `create_vms.yml` playbook with the inventory file and specify the root user name of the Proxmox VM server:

    ansible-playbook create_vms.yml -i inventory --user=root

After running the playbook, virtual machines will be created with configured networks and the specified username and password.

To delete all created virtual machines, use the `delete_vms.yml` playbook:

cssCopy code

    ansible-playbook delete_vms.yml -i inventory --user=root

## HAproxy Configuration

Installation and configuration of HAproxy is done using the `haproxy.yml` playbook:

cssCopy code

    ansible-playbook haproxy.yml -i inventory --user=k8suser

### Manual HAproxy Installation on the Proxy Virtual Machine

    apt install -y haproxy

Open the HAproxy configuration file `/etc/haproxy/haproxy.cfg` and add the following text to the end of the file:

    frontend kubernetes-frontend
    	bind *:6443
    	mode tcp
    	option tcplog
    	default_backend kubernetes-backend
    
    backend kubernetes-backend
    	mode tcp
    	option tcp-check
    	balance roundrobin
    	server k8s-1 192.168.1.230:6443 check fall 3 rise 2
    	server k8s-2 192.168.1.231:6443 check fall 3 rise 2 

Start HAproxy by executing the following 3 commands:

    systemctl start haproxy
    systemctl enable haproxy
    systemctl restart haproxy

## Configuration of k8s-m1 and k8s-m2

On each node, we need to install the following packages:

-   apt-transport-https
-   ca-certificates
-   curl
-   gpg
-   kubelet
-   kubeadm
-   kubectl
-   containerd

And configure the following parameters:

-   overlay
-   br_netfilter
-   net.bridge.bridge-nf-call-iptables = 1
-   net.bridge.bridge-nf-call-ip6tables = 1
-   net.ipv4.ip_forward = 1

All necessary commands for execution are located in the `kubectl_config.sh` file. To configure all nodes, execute the `install_kub.yml` playbook:

    ansible-playbook install_kub.yml -i inventory --user=k8suser

## Cluster Configuration

Connect to the first master node `k8s-m1` and create the cluster. In `--control-plane-endpoint`, specify the IP address of the proxy server:

    sudo kubeadm init --control-plane-endpoint=192.168.1.243:6443 --upload-certs

On the second and subsequent nodes, execute the command to add a control-plane node:

    kubeadm join 192.168.1.243:6443 --token 87oixr.l8kinulpdnnsc1z2 \
    --discovery-token-ca-cert-hash sha256:6ddac82ec84248505e4314ce95c547e9d41cd403a8d3841ed2ae0e8b58a4ea76 \
    --control-plane --certificate-key 2f6df124ab30e6780770ae70ec42a588bdc5bf84494f928aa77fe1f8a1128147

## kubectl Configuration

To configure kubectl to retrieve data from our cluster, execute the following command on one of the cluster servers:

    mkdir -p $HOME/.kube
    sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config

If you want to retrieve cluster data on a local machine, you need to copy the file `/etc/kubernetes/admin.conf` to the local machine in the directory `$HOME/.kube/config`.

## Cluster Status Check

Check the cluster status by executing the command:

    kubectl get nodes 

With this, the setup of a k8s cluster with 2 master nodes is completed.
To make our nodes both master and worker, execute the following command:

    kubectl get no -o name | xargs -n1 kubectl patch --type=json -p '[{
    'op': 'remove',
    'path': '/spec/taints'
    }]'
    kubectl get no -o name | xargs -n1 kubectl patch -p '{
    "metadata": {
    "labels": {
    "node-role.kubernetes.io/control-plane": "true",
    "node-role.kubernetes.io/worker": "true"
    }
    }
    }'

## Cluster Operation Test

As a basic test of the cluster and to understand port forwarding for accessing applications, let's install the nginx web server.

Install the Calico network plugin on the first node:

    kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

Install nginx:

    kubectl create deployment nginx-app --image=nginx

Now, if we execute the command `kubectl get pods`, we will see that we have a deployed nginx-app pod that is running.

At the moment, the pod is only available within the cluster's internal network. To expose the pod externally and make it available on the public network of the cluster, execute the following command:

    kubectl expose deployment nginx-app --type=NodePort --port=80

To view all open ports, execute the command `kubectl get svc`.

The port assigned to our pod is 30809. Check if the pod is accessible via the internal address and port:

    curl http://10.104.151.153:80

To access nginx, we need to add rules to haproxy. Connect to the proxy server. Add the following to the configuration file `/etc/haproxy/haproxy.cfg`:

    frontend nginx
    	bind *:80
    	mode tcp
    	option tcplog
    	default_backend nginx_nodes
    
    backend nginx_nodes
    	mode tcp
    	option tcp-check
    	balance roundrobin
    	server k8s-1 192.168.1.240:30809 check
    	server k8s-2 192.168.1.241:30809 check

Replace port 30809 with the one assigned to your pod and the IP addresses of the servers. Restart the haproxy service for the new rules to take effect:

    systemctl restart haproxy

In the browser, go to [http://192.168.1.243:80](http://192.168.1.243/), replacing 192.168.1.243 with the address of your proxy server, and you will see the standard nginx page.

With this, the setup of the k8s cluster for home use is complete.

## Additional Information

You can edit the playbook according to your needs by adding more virtual servers.

## Manual in other languages [:ukraine:](README-UA.md)