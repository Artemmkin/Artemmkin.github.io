---
layout: post
title: >
    How to build a CI/CD pipeline using Kubernetes, Gitlab CI, and Helm.
tags: [k8s, gitlab, cicd, helm]
---
In today's post I want to share an example of a CI/CD pipeline I created for my test application using very popular nowadays orchestrator Kubernetes (k8s) and Gitlab CI.

### Deploy a Kubernetes cluster
**NOTE:** if you plan to follow my steps make sure to change domain name in the `my-cluster/dns.tf` config file and make appropriate changes in the name server configuration for your domain.

I'm going to use my [terraform-kubernetes](https://github.com/Artemmkin/terraform-kubernetes) repository to quickly deploy a Kubernetes cluster with 3 worker nodes (2 for running my applications and one for Gitlab CI) to Google Cloud Platform.

```bash
$ cd ./my-cluster
$ terraform init
$ terraform apply
```
<!--break-->
If terraform ran successfully, you'll see at the end a [gcloud](https://cloud.google.com/sdk/gcloud/) command which you need to run to configure access to the created cluster with [kubectl](https://kubernetes.io/docs/reference/kubectl/overview/).

```bash
$ gcloud container clusters get-credentials my-cluster --zone europe-west1-b --project example-123456
```


You can check if kubectl is configured correctly by trying to check the server version of k8s:

```bash
$ kubectl version
```

### Configure Kubernetes cluster

Next, I will create namespaces for the services I'm going to run in my cluster. I'll create 2 namespaces for different stages of my example application called `raddit` and one namespace for running Gitlab CI. All of my cluster specific k8s configuration is contained in `my-cluster` directory under subdirectory called `k8-config`:

```bash
$ kubectl apply -f ./k8s-config/env-namespaces
```

I will also create a [storage class](https://kubernetes.io/docs/concepts/storage/storage-classes/) to use in [dynamic volume provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/) for the `mongodb` service which I'm going to deploy later:

```bash
$ kubectl apply -f ./k8s-config/storage-classes/
```

### Deploy Gitlab CI and Kube-Lego

Now, I'm going to deploy [kube-lego](https://github.com/jetstack/kube-lego) for automatic handling of [Let's Encrypt](https://letsencrypt.org/) SSL certificates:

```bash
$ kubectl apply -f ./k8s-config/kube-lego
```

After that, we'll deploy Gitlab CI using [gitlab-omnibus helm chart](https://docs.gitlab.com/ce/install/kubernetes/gitlab_omnibus.html) with slight customizations made. Make sure to change the IP address to the one from terraform output and use your own domain name:

```bash
$ helm init # initialize Helm
$ helm install --name gitlab \
--namespace infra \
--set baseIP=104.155.31.111 \
--set baseDomain=ci.devops-by-practice.fun \
./k8s-config/charts/gitlab-omnibus
```
It's going to take about 5-6 minutes for Gitlab CI to come up. You can check the status of the pods with the command below:
```bash
$ kubectl get pods -n infra
```

### Create a new group of projects

Once Gitlab CI is up and running, it should be accessible at `gitlab.ci.<your-domain>` (use `root` name to log in).

In Gitlab UI, I'll create a new group for my [raddit](https://github.com/Artemmkin/kubernetes-gitlab-example) application:

![400x400](/public/img/k8s-gitlab/grou.png)

Gitlab CI groups allow to group related projects. In our example, `raddit` group will contain the microservices that the raddit application consists of.


Now I will clone a repository with the raddit application:
```bash
$ git clone git@github.com:Artemmkin/kubernetes-gitlab-example.git
```
And create a new project in Gitlab CI web UI for each component of raddit application:

![400x400](/public/img/k8s-gitlab/prjcts.png)


### Describe a CI/CD pipeline for each project

Each component of the raddit application is contained in its own repository and has its own CI/CD pipeline defined in a special `.gitlab-ci-yml` file (which has a special meaning for Gitlab CI) stored in the root of each of the component's directory.

Let's have a look at the **ui** microservice pipeline. Because the pipeline file is long, I'll break it into pieces and comment on each one of them.

First, we define stages in our pipeline and environment variables. The env vars will be set by Gitlab Runner before running each job:

![400x400](/public/img/k8s-gitlab/pipe-ui1.png)

Although Gitlab CI has its own container registry, in this example, I'm going to use Docker Hub, so if you're following along, apart from `GKE info` variables, you'll need to make sure the `CONTAINER_IMAGE` variable is set according to your Docker Hub account name.

Also, don't forget to change the `DOMAIN_NAME` variable.


Now let's go to the pipeline's stages.

In the first (**build**) stage, we build ui application container, tag it by the name of the branch and commit hash and push it to the Docker Hub registry. Then in the **test** stage we can run tests for our application which I have none in this example:

![400x400](/public/img/k8s-gitlab/pipe-ui2.png)

In the following **release** stage, we assign a version tag to the docker images that passed the tests successfully:

![400x400](/public/img/k8s-gitlab/pipe-ui3.png)

The next **deploy** stage is split into 2 jobs. The differences are very small: I don't enable ingress for the service running in `staging` and the deployment to `production` is `manual`. The deployment is done using [Helm](https://helm.sh/) which is a package manager for k8s. You can find helm charts for raddit's microservices in the root of their directories under the `charts` subdirectory.

![400x400](/public/img/k8s-gitlab/pipe-ui4.png)

### Launch pipeline for each service

The pipeline is already described for each service, but you need to change some env vars as I mentioned above in each of the `.gilab-ci.yml` files.

To launch a pipeline for a service, we first need to define some secret variables. Click on **post** project, for example, and go to `Settings -> CI/CD`. Define `CI_REGISTRY_USER` and `CI_REGISTRY_PASSWORD` variables to allow logging to Docker Hub.

Also, define a `service_account` variable to allow Gitlab to deploy to your k8s cluster. To get a value for a `service_account` variable just run `terraform init` and `terraform apply` in the [accounts/service-accounts](https://github.com/Artemmkin/terraform-kubernetes/tree/master/accounts/service-accounts) directory and copy a value from the output.

![400x400](/public/img/k8s-gitlab/secrets.png)

Define the same variables for **ui** and **mongodb** projects.

Now you can push each services folder to the Gitlab repository and the pipeline should start automatically.

![400x400](/public/img/k8s-gitlab/pipeline.png)

### Accessing the Raddit application

In the staging namespace you can access the application by using port-forwarding:

![400x400](/public/img/k8s-gitlab/pf.png)


![400x400](/public/img/k8s-gitlab/staging.png)

and the application deployed into production should be accessible by the domain name via HTTP and HTTPS. Note that it can take up to 5-10 minutes after the first deployment for the application to be reachable by the domain name, because it takes some time to provision a google load balancer:

![400x400](/public/img/k8s-gitlab/prod.png)

### Destroy the playground

After you're done playing with Gitlab CI and k8s and wish to delete the GCP resources that you've created, run the following commands:

```bash
$ helm ls | cut -f1 | tail -n +2 | xargs helm delete --purge # delete all the deployed charts
$ terraform destroy # destroy resources created by terraform
```
