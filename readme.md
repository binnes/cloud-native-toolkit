install minikube

instructions [here](https://minikube.sigs.k8s.io/docs/start/)

brew install minikube

!!!Info
    Ideally you want to know the IP address your cluster will be assigned before it is created.  On MacOS using hyperkit you can do this by looking at file /var/db/dhcpd_leases.  Your machine will be allocated the next available IP address.  If the file does not exist or is empty then you will be assigned 192.168.64.2.  Use this address in the start command to reduce the bumber of manual fixes needed.

minikube start --addons=dashboard --addons=ingress --addons=olm --addons=metrics-server --addons=istio-provisioner --addons=istio --cpus=8 --disk-size=250g --memory=48g --embed-certs --driver hyperkit --insecure-registry 192.168.64.2:5000

verify the cluster ip address with minikube ip

Update catalog-operator Deployment in namespace olm to use image : ```quay.io/operator-framework/olm:v0.17.0```,  which fixes an out of memory crash

Bring up the dashboard with command ```minikube dashboard```, switch to **All namespaces** and then highlight **Workloads** in the side menu.  You shoud see all pods working

### Fix up insecure registry and domain

If you couldn't use the correct IP address for the cluster in the start command, then you need to complete the instructions in this section.  If you were able to use the correct IP address then you can skip to adding the TLS certificates.

On first boot your IP address may not be 192.168.64.2, so run commnd ```minikube ip``` and if the returned address is not 192.168.64.2 then edit file ${HOME}/.minikube/profiles/minikube/config.json and alter the InsecureRegistry value to the ip address returned by the minikube ip command.

Do the same with file ~/.minikube/machines/minikube/config.json - ensue the Insecure Registry entry contains the correct IP address.

Restart minikube with minikube stop then repeat the start command above, replacing the insecure-registry option to the correct IP address for minikube

## Add certificate for TLS - now completed by Ansible

!!!Warning
    The cryptography package needs to be installed before running the ansible playbook
    ```pip install cryptography``` 

export DOMAIN=$(minikube ip).nip.io

openssl genpkey -aes256 -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -pass pass:password123 -out ~/.minikube/certs/ingress_key.pem 

openssl req -newkey rsa:4096 -nodes -out ~/.minikube/certs/ingress_crt.csr -keyout ~/.minikube/certs/ingress_key.pem -subj "/CN=*.${DOMAIN}"

openssl x509 -days 3560 -in ~/.minikube/certs/ingress_crt.csr -out ~/.minikube/certs/ingress_crt.pem -req -sha256 -CA ~/.minikube/certs/ca.pem -passin pass:password123 -CAkey ~/.minikube/certs/ca-key.pem -extensions v3_req -extfile <(printf "\n[ req ]\nattributes = req_attributes\nreq_extensions = v3_req\ndistinguished_name = req_distinguished_name\n[req_distinguished_name]\n[ req_attributes ]\n[ v3_req ]\nsubjectAltName = DNS:*.${DOMAIN}") -set_serial 11

### Add the tls certificate to the cluster as a secret

export SECRET_NAME=$(echo $DOMAIN | sed 's/\./-/g')-tls

kubectl create secret generic tls-ca -n default --from-file=${HOME}/.minikube/certs/rootCA_crt.pem
kubectl label -n default secret tls-ca grouping=garage-cloud-native-toolkit

kubectl create secret tls $SECRET_NAME -n default --cert=${HOME}/.minikube/certs/ingress_crt.pem --key=${HOME}/.minikube/certs/ingress_key.pem
kubectl label -n default secret $SECRET_NAME grouping=garage-cloud-native-toolkit

kubectl create secret tls $SECRET_NAME -n kubernetes-dashboard --cert=${HOME}/.minikube/certs/ingress_crt.pem --key=${HOME}/.minikube/certs/ingress_key.pem

### Add the tlc CA certificate to your local machine / browser

Each OS/browser is different.  On a Mac, generally clicking the cert in the file explorer will open the Keychain Access app and import the certificate.  You should then update the trust settings to Always Trust.  You need to import the ${HOME}/.minikube/certs/rootCA_crt.pem and ${HOME}/.minikube/certs/ingress_crt.pem certificates.

## Add the a local registry service

!!! Warning
    The registry addon does not persist content over a minikube restart, so manually deploying the registry:

    ```kubectl apply -f registry.yaml```
    ```kubectl apply -f  registry-ingress.yaml```

    This creates a persistent volume claim to ensure the data is retained when minikube is stopped then restarted.

!!! Warning
    To push to registry from local command line, then need to add ```"insecure-registries": ["192.168.64.2:5000"]``` to the host docker daemon config - Windows and Mac - this is in the Docker Desktop preferences.  Linux this is in the /etc/docker/daemon.json file.  You need to restart the docker system before the change becomes active

!!! Info
    To access registry information:

    - ```curl 192.168.64.2:5000/v2/_catalog``` will return the current catalog of images
    - ```curl 192.168.64.2:5000/v2/brian/developer-dashboard/tags/list``` will return the tags on image brian/developer-dashboard
    - ```curl 192.168.64.2:5000/v2/brian/developer-dashboard/manifests/latest``` will return the manifest of a specific tagged image

kubectl create secret generic registry-access -n default --from-literal=REGISTRY_URL=registry.$DOMAIN:5000 --from-literal='REGISTRY_NAMESPACE=kube-system'
kubectl label -n default secret registry-access grouping=garage-cloud-native-toolkit

## Expose the dashboard via the Ingress

kubectl apply -f kubernetes-dashboard-ingress.yaml

## Install gitea via Ansible

cd kubernetes-setup

ansible-playbook --connection=local --inventory 127.0.0.1, --limit 127.0.0.1 minicube-playbook.yml

< Update gitea-ingress.yaml to have correct IP address for cluster - minikube ip>

SECRET_NAME=$(echo $DOMAIN | sed 's/\./-/g')-tls

kubectl create secret generic tls-ca -n gitea --from-file=${HOME}/.minikube/certs/rootCA_crt.pem

kubectl create secret tls $SECRET_NAME -n gitea --cert=${HOME}/.minikube/certs/ingress_crt.pem --key=${HOME}/.minikube/certs/ingress_key.pem

kubectl apply -f gitea-ingress.yaml

<restart 2 gitea pods>

### Gitea updates

Edit the gitea secret in the gitea namespace and set the DNS domains to be correct.  For a minikube ip address of 192.168.64.2 the server section of the data content should look like:

```
[server]
APP_DATA_PATH = /data
DOMAIN = gitea.192.168.64.2.nip.io
HTTP_PORT = 3000
PROTOCOL = http
ROOT_URL = http://gitea.192.168.64.2.nip.io
SSH_DOMAIN = git.example.com
SSH_LISTEN_PORT = 22
SSH_PORT = 22
```

The stateful set definition for gitea needs to be modified to add the tls CA cert to the image:
```yaml
spec:
  volumes:
    - name: data
      persistentVolumeClaim:
        claimName: data-gitea-0
    - name: init
      secret:
        secretName: gitea-init
        defaultMode: 511
    - name: config
      secret:
        secretName: gitea
        defaultMode: 420
    - name: ca
      secret:
        secretName: tls-ca
        defaultMode: 420
    - name: default-token-fl727
      secret:
        secretName: default-token-fl727
        defaultMode: 420
  initContainers:
    - name: init
      image: 'gitea/gitea:1.12.6'
      command:
        - /usr/sbin/init_gitea.sh
      resources: {}
      volumeMounts:
        - name: init
          mountPath: /usr/sbin
        - name: config
          mountPath: /etc/gitea/conf
        - name: data
          mountPath: /data
        - name: default-token-fl727
          readOnly: true
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      imagePullPolicy: IfNotPresent
  containers:
    - name: gitea
      image: 'gitea/gitea:1.12.6'
      ports:
        - name: ssh
          containerPort: 22
          protocol: TCP
        - name: http
          containerPort: 3000
          protocol: TCP
      env:
        - name: SSH_LISTEN_PORT
          value: '22'
        - name: SSH_PORT
          value: '22'
      resources: {}
      volumeMounts:
        - name: data
          mountPath: /data
        - name: ca
          mountPath: /tmp
        - name: default-token-fl727
          readOnly: true
          mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      livenessProbe:
        tcpSocket:
          port: http
        initialDelaySeconds: 200
        timeoutSeconds: 1
        periodSeconds: 10
        successThreshold: 1
        failureThreshold: 10
      readinessProbe:
        tcpSocket:
          port: http
        initialDelaySeconds: 5
        timeoutSeconds: 1
        periodSeconds: 10
        successThreshold: 1
        failureThreshold: 3
      lifecycle:
        postStart:
          exec:
            command:
              - /bin/sh
              - '-c'
              - >-
                cp /tmp/rootCA_crt.pem /usr/local/share/ca-certificates/tls.ca.crt &&
                /usr/sbin/update-ca-certificates
      terminationMessagePath: /dev/termination-log
      terminationMessagePolicy: File
      imagePullPolicy: Always
  restartPolicy: Always
  terminationGracePeriodSeconds: 60
  dnsPolicy: ClusterFirst
  serviceAccountName: default
  serviceAccount: default
  nodeName: minikube
  securityContext:
    fsGroup: 1000
  hostname: gitea-0
  subdomain: gitea
  schedulerName: default-scheduler
  tolerations:
    - key: node.kubernetes.io/not-ready
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 300
    - key: node.kubernetes.io/unreachable
      operator: Exists
      effect: NoExecute
      tolerationSeconds: 300
  priority: 0
  enableServiceLinks: true
  preemptionPolicy: PreemptLowerPriority
```

## Gitea setup

!!!Info
    If you want to log onto Gitea as an administrator, look in the gitea-init secret in the gitea namespace.  There you will see the credentials for the admin user: the username is gitea_admin and the password is also visible.  
    
    If you create your own user you can use the admin login to go into the site admin section and promote your user to an administrator.

- Register a new account
- Sign into new account
- Goto settings, then Applications, then Generate Token - copy it and don't loose it
    - pipeline : 1dfb4040dd49269740926a512727dbc474fac90b

### Clone a repo to internal git

- clone repo : git clone https://github.com/.....
- create the repo in the UI of the internal git server (use same name as is used on the public git server)
- git remote remove origin
- git remote add origin https://gitea.192.168.64.2.nip.io/<repo user>/<repo name>.git
- git push -u origin main

## Install che

bash <(curl -sL  https://www.eclipse.org/che/chectl/)
chectl server:deploy --platform minikube
kubectl patch checluster eclipse-che -n che --type merge -p '{ "spec": { "server": {"customCheProperties": {"CHE_LIMITS_USER_WORKSPACE_RUN_COUNT": "-1"} } }}'

You need to add the CA certificate and mark it as trusted to be able to use Che.  This is the same process as used to import the cluster CA and ingress certificates, on a mac simply navigate to where the certificate is using the Finder app, then click on the cert to open in Keychain Access, where it will be imported, then double clicking on it will allow you to change the trust settings.

<default user admin / admin> - need to change password at first login

<todo - how to create new user account in keycloak>

## Install Tekton

kubectl apply --filename https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml

kubectl apply --filename https://storage.googleapis.com/tekton-releases/dashboard/latest/tekton-dashboard-release.yaml

kubectl apply --filename https://storage.googleapis.com/tekton-releases/triggers/latest/release.yaml

kubectl apply -f tekton-ingress.yaml

kubectl create secret generic tls-ca -n tekton-pipelines --from-file=${HOME}/.minikube/certs/rootCA_crt.pem

Update the tekton-pipelines-controller deployment in the tekton-pipelines namespace.  Modify the volumes section to get the tls CA certificate from the secret created above.  Modify the config-registry-cert entry to switch from a configMap to a secret:

```
volumes:
  - name: config-logging
    configMap:
      name: config-logging
      defaultMode: 420
  - name: config-registry-cert
    secret:
      secretName: tls-ca
      defaultMode: 420
```

##  Install rest of cloud native toolkit components

<clone modified ibm-garage-iteration-zero>

cd ibm-garage-iteration-zero
./launch.sh
<wait for docker image to start and command line to appear - you are now running inside the docker container>
./runTerraform.sh -d -a

## Run the example

When creating the name space need to ensure tls-ca secret is in the namespace:

```kubectl create secret generic tls-ca -n dev-bi --from-file=${HOME}/.minikube/certs/rootCA_crt.pem```


## Useful commands

### Remove all completed pods (sucessful or failed)

```kubectl get pods --all-namespaces -o wide | egrep -i 'Error|Completed' | awk '{print $2 " --namespace=" $1}' | xargs kubectl delete pod --force=true --wait=false --grace-period=0```