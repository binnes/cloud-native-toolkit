# Local Cloud Native Toolkit

This repo is based on the [Cloud Native Toolkit](https://cloudnativetoolkit.dev).  The content of this repository extend the toolkit to run on a pure open source stack, based on minikube.

NOTE:  THIS REPOSITORY IS STILL A WORK IN PROGRESS

## Prerequisites

Currently only tested on an Intel based Mac/Linux(Ubuntu) with 16GB of memory or more

This content will currently only run on systems capable of running AMD64 architecture containers, as many of the components are only provided in this architecture

- Docker
- Ansible
- Minikube
- Git Client
- python and pip also need to be installed on the system
- Basic developer tools, to build native components in packages:
  - MacOS : working xcode install with additional command line tools (```xcode-select --install```)
  - Linux : ```sudo apt install -y build-essential```
- Command line tools, kubectl, helm
  - Linux Ubuntu : ```sudo  snap install --classic helm```

## Setup instructions

install Docker
    - Ubuntu Linus instructions -
        ```shell
            snap install docker
            sudo groupadd docker
            sudo usermod -aG docker $USER
            sudo reboot -n
        ```

install ansible, git, python, pip
    - Ubuntu - ```sudo apt-get install -y ansible, python3-pip```

install minikube
    - mac instructions - ```brew install minikube```
    - [Ubuntu linux instructions](https://www.server-world.info/en/note?os=Ubuntu_20.04&p=minikube&f=1)

Additional information [here](https://minikube.sigs.k8s.io/docs/start/)

!!!Info
    Ideally you want to know the IP address your cluster will be assigned before it is created.  On MacOS using hyperkit you can do this by looking at file /var/db/dhcpd_leases.  Your machine will be allocated the next available IP address.  If the file does not exist or is empty then you will be assigned the first available address on the subnet created for the driver used (hyperkit in the example command below).

### Start the Kubernetes cluster

To create the minikube cluster, run the following command, adjusting the CPU and memory settings to match your configuration.  You want to give the stack as much memory as possible to get good working performance.

Start minikube.  The commands below are recommended minimum resource values.  If you have a larger machine you may want to increase CPUs, memory or disk to provide additional resources to minikube:
    mac : ```minikube start --addons=dashboard --addons=ingress --addons=olm --addons=metrics-server --addons=istio-provisioner --addons=istio --cpus=4 --disk-size=50g --memory=12g --embed-certs --driver hyperkit --insecure-registry 192.168.64.2:5000```
    linux : ```minikube start --addons=dashboard --addons=ingress --addons=olm --addons=metrics-server --addons=istio-provisioner --addons=istio --cpus=4 --disk-size=50g --memory=12g --embed-certs --driver kvm2 --insecure-registry 192.168.64.2:5000```

Bring up the dashboard with command ```minikube dashboard &```, switch to **All namespaces** and then highlight **Workloads** in the side menu.  

Wait until all Deployments are complete (Workload status Deployments circle is all green) before continuing.

### Fix up insecure registry and domain

Verify the cluster ip address with ```minikube ip```.

If you didn't use the IP address returned by the ```minikube ip``` command for the insecure-registry option in the start command, then you need to complete the instructions in this section.  If you were able to use the correct IP address then you can skip to adding the TLS certificates.

Edit file ${HOME}/.minikube/profiles/minikube/config.json and alter the InsecureRegistry value to the ip address returned by the ```minikube ip``` command.

Do the same with file ~/.minikube/machines/minikube/config.json - ensue the Insecure Registry entry contains the correct IP address.

Restart minikube with ```minikube stop``` then repeat the start command above, replacing the insecure-registry option to the correct IP address for minikube

!!! Info
    In the rest of the documentation it is assumed that minikube is running on IP address 192.168.64.2.  Wherever you see that address you need to substitute with the IP address of your cluster.

## Run the ansible installer to complete the setup

In a command window:

- clone this git repository: ```git clone https://github.com/binnes/cloud-native-toolkit.git```
- change into the ansible directory: ```cd cloud-native-toolkit/ansible```
- ensure required packages are installed ```pip3 install cryptography```
- install kubernetes collection ```ansible-galaxy collection install community.kubernetes```
- install crypto collection ```ansible-galaxy collection install community.crypto```
- install general collection - for docker ```ansible-galaxy collection install community.general```
- run the playbook: ```ansible-playbook --ask-become-pass minicube-playbook.yml```
  - You will be prompted for your user password - this is because some configuration options need admin privileges

At some point you will be prompted for git credentials for the deployed Gitea server.  The username and password are defined in the vars.yml file to be demo/dem0P4s$.  You can change these if desired.

## Post install steps

A self-signed root certificate has been generated and used to create the ingress TLS certificate.  This needs to be trusted by your browser to be able to access the various applications.

The setup script installs the rootCA certificate to the Mac OS keychain, but the command line tool doesn't appear to setup the trust settings correctly.  Go into the Keychain app (found in Applications -> Utilities).  Select the System keychain and Certificates section.  Find the certificate - named *[IP address]*.nip.io-RootCA.  Change the trust settings to Never Trust, then change back to Always Trust.  When you close the window you should b prompted for your user password, showing that the change has been registered.  This certificate will now be trusted by Safari and Chrome.

In Firefox you need to import the certificate.  The certificate is located in the .minikube/certs folder within your home folder. (~/.minikube/certs) in a file named rootCA_crt.pem.  In Firefox open the Preferences, then select the Privacy & Security settings.  Scroll down until you see the certificates section then press the View Certificates button.  In the Certificate Manager popup window select the Authorities section then Import... Navigate to the .minikube/certs directory within your home directory and select the rootCA_rt.pem file to import.  Select both checkboxes then hit OK to import the certificate.

## Accessing the applications

Minikube runs within a virtual environment on your laptop, so the IP address is not available from outside your system, so by default you need to run on the same machine that minikube is running on.  So launch a browser on the machine and navigate to ```https://dashboard-tools.[ip address].nip.io```.  

    E.g. if ```minikube ip``` returns 192.168.64.2 then the address will be https://dashboard-tools.192.168.64.2.nip.io.  
    
    This is the Cloud Native Developer dashboard.  This has links to the applications the Cloud Native Toolkit installed on your cluster.

!!! Note
    Directories beginning with a . are usually hidden on MacOS.  To see them you can press the **Shift-Command-.** key combination

### Gitea setup

!!!Info
    If you want to log onto Gitea as an administrator, look in the gitea-init secret in the gitea namespace.  There you will see the credentials for the admin user: the username is gitea_admin and the password is also visible.  

    If you create your own user you can use the admin login to go into the site admin section and promote your user to an administrator.

- Register a new account
- Sign into new account
- Goto settings, then Applications, then Generate Token - copy it and don't loose it
  - pipeline : 1dfb4040dd49269740926a512727dbc474fac90b

### Setup che

The default user name and password is admin / admin> - need to change password at first login and don't forget what the new admin password is.

*[todo - how to create new user account in keycloak]*

## Useful commands

### Remove all completed pods (successful or failed)

```kubectl get pods --all-namespaces -o wide | egrep -i 'Error|Completed' | awk '{print $2 " --namespace=" $1}' | xargs kubectl delete pod --force=true --wait=false --grace-period=0```

### manually run the terraform scripts

```docker run -it -e TF_VAR_server_url="https://`minikube ip`:8443" -e TF_VAR_ingress_hostname="`minikube ip`.nip.io" -e TF_VAR_source_control_url="" -e TF_VAR_ibmcloud_api_key="" -v /Users/brian/Cn-Tk/ibm-garage-iteration-zero/terraform:/home/devops/src -v /Users/brian/.kube/config:/home/devops/.kube/config -v /Users/brian/.minikube:/home/devops/.minikube -w /home/devops/src --name cntk-installer quay.io/ibmgaragecloud/cli-tools:v0.10.0-lite /home/devops/src/runTerraform.sh -k -a```

### Install and run local toolkit

- clone repo
- cd into repo directory .\cloud-native-toolkit\ansible
- ```minikube start --addons=dashboard --addons=ingress --addons=olm --addons=metrics-server --addons=istio-provisioner --addons=istio --cpus=8 --disk-size=250g --memory=48g --embed-certs --driver hyperkit --insecure-registry 172.16.171.2:5000```
- ```ansible-playbook --ask-become-pass minicube-playbook.yml```

### to remove all

- ```minikube delete```
- ```rm -rf ~/.minikube```
- ```rm -rf ~/Cn-Tk```
- ```sudo vi /var/db/dhcpd_leases``` - remove all content (or just the entry for the cluster)
- ```docker ps -a``` - remove all exited containers
- remove the certificate 172.16.171.2.nip.io-RootCA from the trusted authorities (Keychain app on Mac)
