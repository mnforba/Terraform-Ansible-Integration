# Terraform-Ansible-Integration
The most simplified integration of Ansible and Terraform

In this demonstration, I have tried to integrate ansible and terraform by coordinating Terraform managed nodes with Ansible control nodes. For this, I will launch an AWS instance using terraform and then will use Ansible to configure that instance to deploy our webpage that will automatically open up after deployment.

## Creating AWS EC2 instance using Terraform: 
To begin with the setup we use terraform code to create the infrastructure on AWS. We create an AWS EC2 instance and an EBS volume attached to it.
After the infrstructure is created using Terraform all the information regarding the provisioned resource is saved in its terraform.tfstate file. This state file is used to retrieve the IP of the launched instance which is passes in the Ansible inventory file. This inventory file holds the information regarding its managed node where Ansible needs to install necessary applications.

## Create Dynamic Ansible inventory file and copy to remote Ansible Control node: 
To begin with the setup we use terraform code to create the infrastructure on AWS. We create an AWS EC2 instance and an EBS volume attached to it.
After the infrstructure is created using Terraform all the information regarding the provisioned resource is saved in its terraform.tfstate file. This state file is used to retrieve the IP of the launched instance which is passes in the Ansible inventory file. This inventory file holds the information regarding its managed node where Ansible needs to install necessary applications.
 ##### IP of aws instance copied to a file ip.txt in local system
    `resource "local_file" "ip" {  
         content  = aws_instance.demo1.public_ip
	 filename = "ip.txt"
    }
 ##### connecting to the Ansible control node using SSH connection
    resource "null_resource" "nullremote1" {
    depends_on = [aws_instance.demo1]
    connection {
        type     = "ssh"
	user     = "root"
	password = "${var.password}"
	   host  = "${var.host}"
      }

   ##### copying the ip.txt file to the Ansible control node from local system:
    `provisioner "file" {
          source    = "ip.txt"
	  destination = "/root/ansible_terraform/aws_instance/ip.txt
    }
  }

   ### `On Master and Worker:`
1. Perform all the commands as root user unless otherwise specified
 
   Install, Enable and start docker service.
   Use the Docker repository to install docker.
   > If you use docker from CentOS OS repository, the docker version might be old to work with Kubernetes v1.13.0 and above

   ```sh
   yum install -y -q yum-utils device-mapper-persistent-data lvm2 > /dev/null 2>&1
   yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo > /dev/null 2>&1
   yum install -y -q docker-ce >/dev/null 2>&1
   ```
1. Start Docker services 
   ```sh
   systemctl enable docker
   systemctl start docker
   ```
1. Disable SELinux
   ```sh
   setenforce 0
   sed -i --follow-symlinks 's/^SELINUX=enforcing/SELINUX=disabled/' /etc/sysconfig/selinux
   ```
1. Disable Firewall
   ```sh
   systemctl disable firewalld
   systemctl stop firewalld
   ```
1. Disable swap
     ```sh
     sed -i '/swap/d' /etc/fstab
     swapoff -a
    ```
1. Update sysctl settings for Kubernetes networking
   ```sh
   cat >> /etc/sysctl.d/kubernetes.conf <<EOF
   net.bridge.bridge-nf-call-ip6tables = 1
   net.bridge.bridge-nf-call-iptables = 1
   EOF
   sysctl --system
   ```
## Kubernetes Setup
1. Add yum repository for kubernetes packages 
    ```sh
    cat >>/etc/yum.repos.d/kubernetes.repo<<EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
    enabled=1
    gpgcheck=1
    repo_gpgcheck=1
    gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg
            https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
    EOF
    ```
1. Install Kubernetes
    ```sh
    yum install -y kubeadm-1.15.6-0.x86_64 kubelet-1.15.6-0.x86_64 kubectl-1.15.6-0.x86_64
    ```
1. Enable and Start kubelet service
    ```sh
    systemctl enable kubelet
    systemctl start kubelet
    ```
## `On Master Node:`
1. Initialize Kubernetes Cluster
    ```sh
    kubeadm init --apiserver-advertise-address=<MasterServerIP> --pod-network-cidr=192.168.0.0/16
    ```
1. Create a user for kubernetes administration  and copy kube config file.   
    ``To be able to use kubectl command to connect and interact with the cluster, the user needs kube config file.``  
    In this case, we are creating a user called `kubeadmin`
    ```sh
    useradd kubeadmin 
    mkdir /home/kubeadmin/.kube
    cp /etc/kubernetes/admin.conf /home/kubeadmin/.kube/config
    chown -R kubeadmin:kubeadmin /home/kubeadmin/.kube
    ```
1. Deploy Calico network as a __kubeadmin__ user. 
	> This should be executed as a user (heare as a __kubeadmin__ )
    
    ```sh
    sudo su - kubeadmin 
    kubectl create -f https://docs.projectcalico.org/v3.9/manifests/calico.yaml
    ```

1. Cluster join command
    ```sh
    kubeadm token create --print-join-command
    ```
## `On Worker Node:`
1. Add worker nodes to cluster 
    > Use the output from __kubeadm token create__ command in previous step from the master server and run here.

1. Verifying the cluster
    To Get Nodes status
    ```sh
    kubectl get nodes
    ```
    To Get component status
    ```sh
    kubectl get cs
    ```
