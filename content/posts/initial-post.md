---
title: "Initial Post"
date: 2019-05-27T20:48:00+02:00
draft: true
---

## Introduction

With the rising popularity of serverless computing there is a growing demand for Cloud-provider independent serverless solutions allowing teams to benefit from modern architectural software patterns regardless of their technical foundation. Luckily there’s a wide array of solutions to pick from nowadays. The [Cloud Native Computing Foundation][cncf] does a great job mapping out all of them in their [CNCF landscape][cncf-landscape]. A good starting point to dive into the serverless ecosystem is [OpenFaaS][tool-openfaas] - a user-friendly project with an active open-source community.

## What's the goal of this tutorial?

This tutorial explains how to setup OpenFaaS on a DigitalOcean managed Kubernetes cluster using Terraform. The resulting cluster will be using Traefik as an Ingress Controller, ExternalDNS for automatic DNS configuration, and Linkerd as a service mesh. After finishing the tutorial you will be able to automatically provision a Kubernetes cluster with these tools installed. You will be able to add functions to your OpenFaas installation and expose them to the internet. You will also have a better understanding of the tools in use.

The goal of this tutorial is not to set up all components in a production-ready fashion. Instead it will give you an environment with reasonable defaults that allows you to further dive into Kubernetes and the applications running on Kubernetes.

Note: The Terraform modules used in this example do most of the work automatically. That is the magic of infrastructure-as-code after all. The tutorial will guide you through the different Terraform steps and explain how they build on each other. In the end you should be able to derive how to use these tools for your own use-cases (e.g. with different platform providers or to deploy your own applications).

## Prerequisites

To get started with the tutorial, you will need:

* A Digital Ocean account and a Digital Ocean personal access token. To create a token, follow these instructions.
* A `git` client to clone the repository with the required Terraform modules.
* `terraform`, `kubectl`, `linkerd` and `helm` CLI installed on your local machine. You will use terraform to provision resources in DigitalOcean and then to orchestrate the usage of `kubectl`, `linkerd` and `helm`. Of course, all of this is done via Terraform, so all you have to do is just provide valid parameters initially.
* A fully registered domain name. The required DNS entries will automatically be configured during the setup.
* A general understanding of the core Kubernetes concepts and tools used in this tutorial as this guide won’t introduce them.


## Step 1 - Setting up the working environment

All sources used in this tutorial are available on Github. Begin by cloning the repository to your local machine:

    git clone https://github.com/pecan-pie/trainingscenter.git

The repository contains all required Terraform modules to setup the Kubernetes cluster described above. Start by entering the following command:

    terraform init

This will initialize the Terraform working directory and download the required Terraform providers for your operating system. 

> The script has been tested with Ubuntu 18.04.1 LTS running on the Windows Subsystem for Linux (WSL).

## Step 2 - Initializing all infrastructure components 

After initializing Terraform we can now setup the infrastructure components using the Terraform modules. The following sections will cover each step that happens when you run Terraform.

### Running Terraform

By entering the following command you can list all components that will be created by Terraform. You can confirm the creation off all components by typing `yes` when asked or add the `-auto-approve` flag to the command to accept creation automatically.

    terraform apply -var do_token=${DO_PAT} -var acme_mail=recipient@sample.com -var domain=sample.com

You can also use the command `terraform plan` to create an execution plan. This is a convenionet way to check which components will be affected by Terraform before you check changes in to version control.

> **Note:** These commands require the environment variable `DO_PATH` to contain a valid access token for your DigitalOcean account, as well as the domain name and an email adress to use for the Let's Encrypt challenge. The Terraform modules expect that DigitalOcean will be used for DNS registration.

After telling Terraform to set up your cluster you have some time to dive into each module to check what is actually happening in the process. Let's start at the top:

### Initializing the DigitalOcean Provider

> This section refers to: `do_provider.tf`

To get everything rolling we need to initialize the Terraform [DigitalOcean provider][do-provider]. This provider is used by other resources built on DigitalOcean infrastructure (e.g. resources to set up the Kubernetes cluster and DNS entries).

### Provisioning a Digital Ocean Kubernetes cluster

> This section refers to: `do_kubernetes.tf`

The first component to set up is the Kubernetes cluster. This is done using the [DigitalOcean Kubernetes resource][do-res-kubernetes]. In case you require other parameters, e.g. a different region, you can adapt these values to your liking:

    name    = "trainingscenter"
    region  = "fra1"
    version = "1.13.5-do.1"
   
In case you need more or better performing nodes in your cluster, you can adjust the parameters for the `node_pool` in this file as well:

    node_pool {
      name       = "worker-pool"
      size       = "s-2vcpu-4gb"
      node_count = 3
    }

Finally, this is also where the configuration file for the new Kubernetes cluster is written to disk. The target location for the config is `contexts/kube-cluster-<clusterName>.yml` within the working directory. 

### Creating DigitalOcean Domain entry

> This section refers to: `do_domain.tf`

The `digitalocean_domain` resource creates the necessary domain item on DigitalOcean. 

    resource "digitalocean_domain" "default" {
      name = var.domain
    }

This item holds the DNS entries required for domain resolution and SSL verification. We will dive into this topic later in the Traefik and ExternalDNS setup. 

### Setting up Linkerd

> This section refers to: `linkerd.tf`

After the Kubernetes cluster has been set up succesfully we will add [Linkerd][tool-linkerd] as service mesh. To quote the Linkerd documentation it "makes running services [...] safer by giving you runtime debugging, observability, reliability, and security — all without requiring any changes to your code". Linkerd does this by injecting sidecar containers with your application pods which intercept incoming and outgoing communication. These sidecar containers add features such as encryption and telemetry to your applications. The [Linkerd architecture documentation][linkerd-architecture] does a great job explaining these concepts in detail.

We add Linkerd functionality to our Kubernetes cluster by making heavy use of [auto-injection][linkerd-auto-inject] which can be enabled by annotating namespaces accordingly. This means that all pods deployed in our cluster will automatically be provided with a Linkerd sidecar container. The installation of Linkerd is based on the [documentation][linkerd-setup]:

    linkerd install --proxy-auto-inject --skip-outbound-ports 4222 --skip-inbound-ports 4222 | kubectl apply -f -

Skipping port 4222 is necessary as OpenFaaS uses NATS which is a server-speaks-first protocol and [requires some extra configuration][linkerd-nats] as such. 

After adding the Linkerd components to our cluster we will annotate the default namespace to enable auto-injection as well. This causes Linkerd to add a proxy sidecar container to each pod in this namespace. 

    kubectl annotate namespace default linkerd.io/inject=enabled --overwrite

Now we just need to wait until all components are ready.

    kubectl wait --for=condition=Ready pods --all --namespace linkerd --timeout=360s

After these commands are finished running Linkerd is ready to use and we can move on to setting up actual applications on our Kubernetes cluster.  

> **Note:** The commands quoted in this section are abbreviated for readability.

### Preparing Helm

> This section refers to:  `helm_setup.tf`

To install applications in our cluster we are using Helm as it is a popular choice for package mangement in the Kubernetes ecosystem. Unfortunately the Terraform Helm provider currently only supports a Tiller-based approach, i.e. running a cluster-side component that manages the lifecycle of our deployed applications. This approach will change with Helm Version 3 which is not stable as of today. 

Using Terraform we will create a Tiller service account in the default namespace which will handle deployments in all namespaces. Please note, in this tutorial we are lenient with access rights to keep the example concise. In a production environment you have to implement minimal privileges for Tiller service accounts.

    resource "kubernetes_service_account" "tiller" {
        metadata {
            name      = "tiller"
            namespace = "default"
        }
        ...
    }

    resource "kubernetes_cluster_role_binding" "tiller" {
        role_ref {
            api_group = "rbac.authorization.k8s.io"
            kind      = "ClusterRole"
            name      = "cluster-admin"
        }

        subject {
            kind      = "ServiceAccount"
            name      = "tiller"
            namespace = "default"
        }
        ...
    }

After creating the service account for Tiller we can initialize the Terraform Helm provider which will be used for the deployment of applications in the next few steps.

    provider "helm" {
        service_account = "tiller"
        namespace = "default"
        install_tiller  = true
        ...
    }

After initializing the Helm provider we can finally deploy Traefik and ExternalDNS. The order in which we do this is not relevant as both applications are independent from each other.

### Installing Traefik

> This section describes the purpose of the following files: `helm_traefik.tf`, `traefik-values.yml`

LoadBalancer


### Installating ExternalDNS

> This section describes the purpose of the following files: `helm_externaldns.tf`

### Quick Intermission

That's all the basic infrastructure components you need to get started. You have set up a Kubernetes cluster automatically and repeatably. The cluster has an Ingress provider which automatically creates routes to your applications and secures them using TLS. All components within your cluster are being meshed which makes them use mutual TLS and gives you detailed insights into the communication between them. 

This is probably not a bad time to take a break, as this were quite a number of different tools already. However, all this effort only really makes sense when you deploy your own functions and services to the cluster. This is what we are going to tackle in the next chapter.

## Step 3 - Quick dive into OpenFaaS

### Initializing OpenFaaS

> This section refers to:  `helm_openfaas.tf`, `openfaas-values.yml`

[Blog Entry](colorise-cats)

`kubectl get secret basic-auth -o jsonpath='{.data.basic-auth-password}' -n openfaas | base64 --decode -`

## Cleaning up after you're done

`terraform destroy -var do_token=${DO_PAT} -var acme_mail=recipient@sample.com -var domain=sample.com`

## Closing remarks

[cncf]: https://www.cncf.io/
[cncf-landscape]: https://github.com/cncf/landscape
[colorise-cats]: colorise-your-cats-with-openfaas "colorise-cats"
[do-provider]: https://www.terraform.io/docs/providers/do/index.html
[do-res-kubernetes]: https://www.terraform.io/docs/providers/do/r/kubernetes_cluster.html
[tool-openfaas]: https://www.openfaas.com/
[tool-kubectx]: https://github.com/ahmetb/kubectx
[tool-linkerd]: https://linkerd.io/
[linkerd-auto-inject]: https://linkerd.io/2/features/proxy-injection/
[linkerd-setup]: https://linkerd.io/2/getting-started/
[linkerd-nats]: https://linkerd.io/2/features/protocol-detection/
[linkerd-architecture]: https://linkerd.io/2/reference/architecture/
