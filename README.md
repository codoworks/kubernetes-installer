# Codoworks Kubernetes Installer

```
   ___          _                         _        
  / __\___   __| | _____      _____  _ __| | _____ 
 / /  / _ \ / _  |/ _ \ \ /\ / / _ \| '__| |/ / __|
/ /__| (_) | (_| | (_) \ V  V / (_) | |  |   <\__ \
\____/\___/ \__,_|\___/ \_/\_/ \___/|_|  |_|\_\___/ cli.

```

## Introduction
This script is built using Ansible. It is intended to install and configure a bare-metal kubernetes cluster on self-hosted VMs.

It is designed to work with a privileged user named `ansible`. Luckily, this script will take care of the new user creation, however, it is important to keep in mind that we need to use a different privileged user (e.g. `root`) to complete that one step of creating a user named `ansible` before the rest of the script can be executed. 

> Basically, any playbook that starts with number `0` should be executed as `root`.\
> to do that, simply add `-u root` to your command.

## Script Requirements

There's absolutely nothing that needs to be done on any of the VMs.

On the control workstation only, you'll need to install a few tools.
1. `ansible` - which by default downloads `ansible-playbook` cmd. you can install that using:
```
brew install ansible
```
2. `sshpass` - an add-on to help Ansible authenticate via password.
```
brew install sshpass
```

## Setup Ansible - latest

Before you can execute any scripts, you'll need to authenticate the machine you're using with the servers you intend to manage. As a first time setup, you'll need to manually login (`ssh`) to each VM and accept the fingerprint recognition. When that's done, follow the next steps:

> To authenticate, all you need to do is `ssh username@IP-address` and accept the prompts.

### Mandatory steps

1. on the control workstation, create a new ssh key
```
ssh-keygen -t ed25519 -C 'Ansible'
```
store the newly generated private and public key in a folder named `./playbooks/keys`.\
you should end up with 2 files as follow\
`./playbooks/keys/ansible`\
`./playbooks/keys/ansible.pub`

2. on the control workstation, create an inventory file in the root directory of this repo, containing all IP addresses of all VMs as follow:
```
[initializing_master_node]
10.0.1.11

[master_nodes]

[worker_nodes]
10.0.1.12
10.0.1.13
10.0.1.14
10.0.1.15

[ha_proxy_node]
10.0.1.10

[test_nodes]

```

3. on the `initializing_master_node`, make sure you have ports 6443, 179, 10250 open
> It's very important to have ports 6443 and 179 open.\
> Port 6443 is needed by kubelet to communicate with Kubernetes API.\
> Port 179 is needed by Calico to communicate with other Pods.\
> Port 10250 is needed by the Metrics Server.

### Optional steps

This step is optional, the purpose of it is to prevent the need for manual password key-in. it's optional because you'd only need to key-in the password once for the root user so it doesn't affect much. 

- copy the newly created public key (ending with .pub) and paste it inside `.ssh/authorized_keys` in each of the VMs.

## Test your setup
Once you've setup the inventory list and ssh key, you can now test your connections.\
There are two ways to test your setup, either using `ansible` cmd or using a playbook. 

1. The first way involves `ansible` directly like:
```
ansible all -m ping --ask-pass -u <username | i.e. root>
```
You should see a result similar to the following: 
```
...
10.0.1.11 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12"
    },
    "changed": false,
    "ping": "pong"
}
10.0.1.12 | SUCCESS => {
    "ansible_facts": {
        "discovered_interpreter_python": "/usr/bin/python3.12"
    },
    "changed": false,
    "ping": "pong"
}
```

2. The second way involves using `ansible-playbook` to achieve the same result as #1 using:
```
ansible-playbook playbooks/00_ping_inventory.yml --ask-pass -u <username | i.e. root>
```

## Create user account
1. make sure to set the user to `root` in `./ansible.cfg`, then run the following to create a new user named `ansible`:
```
ansible-playbook playbooks/01_new_user.yml -k --ask-become-pass -u <username | i.e. root>

OUTPUT:
SSH password: 
BECOME password[defaults to SSH password]: 
Username for new account to be created in remote server?:
```
Key in `ansible` when prompted for a name by the above command. Once completed, you can proceed to install Kubernetes.

To test your setup, you can re-run the ping command without a user parameter, which will fall back to the default user the script is designed to use:
```
ansible-playbook playbooks/00_ping_inventory.yml
```

## Install Kubernetes

> As a rule of thumb, you may wanna update all packages before you start.\
> To do so, just run `ansible-playbook playbooks/10_update_packages.yml`.

1. You can install kubernetes using the following scripts
```
ansible-playbook playbooks/20_install_kubernetes.yml  
ansible-playbook playbooks/21_init_cluster.yml  
```
At this point, you should have a Kubernetes cluster fully up and running. you can verify your setup by logging into (`ssh`) the initializing_master_node and running:
```
kubectl get nodes 

OUTPUT SAMPLE:
NAME          STATUS   ROLES           AGE   VERSION
k8s-node-01   Ready    control-plane   50m   v1.32.2
k8s-node-02   Ready    <none>          35m   v1.32.2
k8s-node-03   Ready    <none>          35m   v1.32.2
k8s-node-04   Ready    <none>          35m   v1.32.2
k8s-node-05   Ready    <none>          35m   v1.32.2    
```
And to make sure the cluster is in a good condition, you can run:
```
kubectl get pods --all-namespaces

OUTPUT SAMPLE:
NAMESPACE     NAME                                       READY   STATUS    RESTARTS   AGE
kube-system   calico-kube-controllers-77969b7d87-pmbbf   1/1     Running   0          51m
kube-system   calico-node-f254h                          1/1     Running   0          36m
kube-system   calico-node-jj75j                          1/1     Running   0          51m
kube-system   calico-node-l269q                          1/1     Running   0          36m
kube-system   calico-node-m7tbg                          1/1     Running   0          36m
kube-system   calico-node-txg2b                          1/1     Running   0          36m
kube-system   coredns-668d6bf9bc-2mvgn                   1/1     Running   0          51m
kube-system   coredns-668d6bf9bc-d6g69                   1/1     Running   0          51m
kube-system   etcd-k8s-node-01                           1/1     Running   0          51m
kube-system   kube-apiserver-k8s-node-01                 1/1     Running   0          51m
kube-system   kube-controller-manager-k8s-node-01        1/1     Running   0          51m
kube-system   kube-proxy-9qqnq                           1/1     Running   0          36m
kube-system   kube-proxy-dsdvc                           1/1     Running   0          36m
kube-system   kube-proxy-htqmn                           1/1     Running   0          36m
kube-system   kube-proxy-jngm6                           1/1     Running   0          36m
kube-system   kube-proxy-snssn                           1/1     Running   0          51m
kube-system   kube-scheduler-k8s-node-01                 1/1     Running   0          51m
```

2. You can now install additional pods 
```
ansible-playbook playbooks/30_install_metrics_server.yml  
ansible-playbook playbooks/31_install_nginx_ingress.yml  
```


