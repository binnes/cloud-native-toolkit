# Where to run the Cloud Native Toolkit

The [Cloud Native Toolkit](https://cloudnativetoolkit.dev) is typically run within [Open Shift running on the IBM Cloud](https://www.ibm.com/cloud/openshift), however this is not always ideal for learning.  Sometime a developer wants a local option, where everything can run on their own hardware, maybe where connectivity isn't always guaranteed.  There are other times where a developer may want to dig a little deeper, so wants an isolated environment, where they can experiment without fear of breaking other members of their team, so again want an isolated environment where they have admin permissions.

Having access to a longer term, no cost environment can also give a developer time to do deeper learning

This is why this project exists.  To take the Cloud Native toolkit and allow it to be run in an isolated environment with lower resources than would typically be available in a cloud environment.

## Low Resources environment

One of the challenges of working with a local install is that the software stack is designed to run in a cloud environment, where resources are more abundant, and typically designed to serve a large user population.  This can make it challenging to replicate on a laptop or developer workstation.

There are also capabilities which are assumed in a cloud environment, which may not exist when running locally, such as inbound connectivity.

This project aims to provide 'as close as is possible' developer experience to the primary target environment on the cloud, but running locally on a laptop or developer workstation.

### Minimum Environment

The minimum resources needed to run the toolkit on a laptop or developer workstation is a machine with 16GB memory.  The target is to support modern versions of Linux, MacOS and Windows operating systems, but currently only MacOS has been tested

The minimum environment run on top of Minikube rather than OpenShift, as Minikube reduces memory requirements at the cost of the richer cloud platform and the enhanced developer experience provided by OpenShift.

### Enhanced Environment

A better developer experience can be obtained where a laptop of developer workstation has more memory available, 24GB or greater.  This can then use [Code Ready Containers - crc](https://developers.redhat.com/products/codeready-containers/overview) or the [Open Source community distribution based version - okd](https://www.okd.io/crc.html)

!!! Note
    Use the [Cloud Native Toolkit](https://cloudnativetoolkit.dev) instructions to setup this environment.  The project has not yet created the assets to install a stand-alone version of the toolkit based on OpenShift (crc/okd).

## Challenges of running locally

Running the environment locally does pose some challenges to deliver a good developer workflow:

### DNS - Name resolution

When you deploy workloads on a cloud your browser and clients of the workloads need to be able to reach the workloads.  Facilities like the Ingress controller within the cloud infrastructure will route traffic within the cloud, but a browser still needs to be able to get to the cloud.  With public cloud hosted services, the Internet DNS service enables name resolution to an IP address, but such services are not available for local kubernetes clusters, unless you setup and configure your own DNS server or manually configure mappings for all services on your workstation.

To overcome this issue the [**nip.io**](https://nip.io) name resolution service is used by the project.  This does require an outbound internet connection to work, but solves name resolution issues for workloads deployed to a local kubrnetes cluster.  

The service works by including the cluster IP address in the URL.  Service URLs need to be in the form of [service name].[ip address].nip.io and they will resolve the to IP address included in the URL.  

This project sets the cluster domain to [minikube ip address].nip.io, so all ingress hostnames will have a valid nip.io format and all will resolve to the kubernetes IP address.

If you need to run in a disconnected environment then a DNS server, such as **dnsmasq** needs to be installed and configured in your local network/workstation.

!!! note
    This is on the todo list to provide instructions to implement local DNS

### TLS Certificates

Secure communication is provided by TLS based on certificates and a set of trusted certificate authorities.  The public certificates for these trusted certificate issuing authorities are included in operating systems and some browsers.  When working with services accessible over this internet then you can configure a cluster to dynamically request certificates or purchase certificates for use within an enterprise.  However, to use free certificate authorities, such as [LetsEncrypt](https://letsencrypt.org) you need to verify you control the domain you are requesting a certificate for, so need to setup a server to accept incoming requests for that domain, as registered in public DNS servers.

With a local setup it is not always possible to set this up and where it is possible it is often not trivial.  Having to use a traffic forwarding service, dynamic DNS forwarder, etc.  then having to configure your service provider router or modem to forward traffic within your local network.  If working at a hotspot or with certain internet service providers it is not possible to accept inbound traffic, so an alternate solution is needed to provide SSL/TLS certificates.

This project creates a self-signed root Certificate Authority (CA) certificate, then issues a wildcard server certificate to the cluster *[minikube IP address].nip.io, which is signed by the self-signed root CA certificate.  You need to add and trust the public certificate of the self-signed root CA to your OS/browser then all services offered by the local cluster will be trusted and you won't receive any browser warnings.  Any application or services that need to access a cluster service will also need to have the root CA public certificate added to their host OS to allow them to communicate securely without reporting certificate authorisation errors.

This approach works for disconnected or connected working, once the CA root public certificate is trusted.

### Inbound connectivity to trigger pipelines

The Cloud Native Toolkit relies heavily on the source control system, usually github.  However, part of the workflow requires git hooks to fire and initiate actions within the development environment.  When running on public cloud infrastructure github.com is able to connect with the required services running the developer workflow, but in a local environment this is not possible.  

To provide the integrated developer workflow this project installs a private git service on the cluster, so git hooks can be configured to initiate workflows within the local cluster.  

The current choice of private git service is [Gitea](https://gitea.io), but the Cloud Native Toolkit has just added support for [Gogs](https://gogs.io), so it may be preferable to adopt that in the future?
