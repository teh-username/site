---
title: Gitlab Runner using Kubernetes Executor
date: 2019-01-13 20:30:47
tags:
  - kubernetes
  - aws
  - gcp
  - gitlab
---

I was recently involved with setting up a Gitlab pipeline. Imagine this, you still have the usual Gitlab Runner connected to your repository, but instead of running the jobs in a VM, the Gitlab Runner talks to a Kubernetes cluster and spins up the job there instead!

This setup isn't anything new as there are some entries floating around that already tackles this setup, like this [one](https://edenmal.moe/post/2017/GitLab-Kubernetes-Running-CI-Runners-in-Kubernetes/) (visualized below):

```
                       +---------------------------------------+
                       | Kubernetes Cluster                    |
                       |                                Pod    |
                       |                             +-------+ |
                       |                       ----> | Job 1 | |
+--------+             | +-------------+ -----/      +-------+ |
| Gitlab | ----------> | |Gitlab Runner|                       |
+--------+   Trigger   | +-------------+ -----\      +-------+ |
             Runner    |       Pod             ----> | Job 2 | |
                       |                             +-------+ |
                       |                                Pod    |
                       |                                       |
                       +---------------------------------------+
```

What is different with our setup though is that we yanked the Gitlab Runner out of the cluster from the setup above and placed it on a separate machine. Here's a few reasons why you'd want to do that:

* You already have a pre-existing Gitlab Runner running on some machine and you just want add the cluster as another executor.

* There's a possibility that you might be using other executors in the future so you'd like to keep your setup as flexible as possible and not be locked in with your runner inside a cluster.

Our sample setup will look like (to keep things neutral, we'll be using both AWS and GCP):

```
                                                          +-------------------------+
                                                          | Google Cluster          |
                                                          |                         |
                        +-------------------+             |                  Pod    |
                        | Amazon Lightsail  |             |    Pod        +-------+ |
 +--------+             |                   |             | +-------+     | Job 3 | |
 | Gitlab | ----------> | +---------------+ | ----------> | | Job 1 |     +-------+ |
 +--------+   Trigger   | | Gitlab Runner | |   Schedule  | +-------+               |
              Runner    | +---------------+ |     Job     |          +-------+      |
                        +-------------------+             |          | Job 2 |      |
                                                          |          +-------+      |
                                                          |             Pod         |
                                                          +-------------------------+
```

For the recreation, we aren't going to use any cloud provider specific magic so you should be able to migrate this setup with whatever infrastructure you currently have.

## Kubernetes Cluster

To kick things off, let's spin up a Kubernetes Cluster using [GKE](https://cloud.google.com/kubernetes-engine/). We'll just leave all settings to default except the name. Don't forget to configure your `kubectl` to the cluster once it is up and running!

{% asset_img gke_ui.png GKE UI %}

### Optional Step (only applicable if your cluster is in GKE):

Since we'll be utilizing RBAC related stuff later, we need to run the following command:

``` bash
kubectl create clusterrolebinding cluster-admin-binding --clusterrole cluster-admin --user [USER ACCOUNT]
```

Where `[USER ACCOUNT]` is the email you are using for GCP. You can find more about RBAC for GKE [here](https://cloud.google.com/kubernetes-engine/docs/how-to/role-based-access-control).

The next few steps involves getting some cluster related information that is needed when we setup the Gitlab Runner later.

### CA Certificate

The CA Certificate is required since we'll be calling the API over HTTPS and the TLS certificate associated with the endpoints is self-signed by the cluster.

To get the cluster CA certificate, run the following command:

``` bash
kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d
```

It should look roughly like:

```
-----BEGIN CERTIFICATE-----
...snip...
-----END CERTIFICATE-----
```

### Roles

We'll now be creating the Service Account that the runner will use along with the associated roles needed to spin up pods. Our setup follows the Principle of Least Privilege so the roles only lists the absolute minimum API actions required for the setup to work. The Helm template can be found [here](https://gitlab.com/teh-username/gitlab-k8s-proxy-demo). Just run:

```
kubectl apply -f <(helm template runner)
```

You can tweak the settings by altering the `values.yaml` file.

### Service Account Bearer Token

Next, we'll need to pass a credential of sort to our Gitlab Runner so it can authenticate itself as the service account we just made. There are a [variety of ways](https://kubernetes.io/docs/reference/access-authn-authz/authentication) to do this but we'll opt for the [Service Account Tokens](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#service-account-tokens) route. We are basically going to get our service account's bearer token and setup Gitlab Runner to use it. To retrieve the token, run the following command:

``` bash
kubectl get secrets -o jsonpath="{.items[?(@.metadata.annotations['kubernetes\.io/service-account\.name']=='SERVICE_ACCOUNT_NAME')].data.token}" | base64 -d
```

Don't forget to adjust the `SERVICE_ACCOUNT_NAME` to what you used during the role creation step.

## Gitlab Pipeline

Up next, let's create a Gitlab repository to setup our pipeline. Just commit a file named `.gitlab-ci.yml` with the following contents:

``` yaml
test:
  image: alpine:latest
  tags:
    - k8s-runner
  script:
    - echo "hello world!"
```

We'll also need to take note of our CI/CD registration token as shown below:

{% asset_img gitlab_ci.png Gitlab CI/CD settings %}

## Gitlab Runner

We'll now be spinning up an Amazon Lightsail instance (Ubuntu 18.04) where our Gitlab Runner will reside. Make sure to add the following as the launch script for your machine:

``` bash
#!/bin/bash

# Taken from: https://docs.gitlab.com/runner/install/linux-repository.html
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | bash

cat > /etc/apt/preferences.d/pin-gitlab-runner.pref <<EOF
Explanation: Prefer GitLab provided packages over the Debian native ones
Package: gitlab-runner
Pin: origin packages.gitlab.com
Pin-Priority: 1001
EOF

apt-get install gitlab-runner

# Create ca.crt file
mkdir /home/ubuntu/k8s && touch /home/ubuntu/k8s/ca.crt

# Dump contents to file
cat > /home/ubuntu/k8s/ca.crt <<EOF
-----BEGIN CERTIFICATE-----
CA_CERT
-----END CERTIFICATE-----
EOF

# Register the runner
gitlab-runner register \
  --non-interactive \
  --url "GITLAB_URL" \
  --registration-token "GITLAB_CI_TOKEN" \
  --executor "kubernetes" \
  --description "kubernetes-runner" \
  --tag-list "k8s-runner" \
  --kubernetes-host "https://KUBERNETES_URL" \
  --kubernetes-ca-file "/home/ubuntu/k8s/ca.crt" \
  --kubernetes-bearer_token "SERVICE_ACCOUNT_BEARER_TOKEN"
```

Take note of the following variables in the script:

* `CA_CERT` - The CA certificate we've retrieved from the cluster
* `GITLAB_URL` - URL specified by Gitlab in the CI/CD Runners settings page
* `GITLAB_CI_TOKEN` - The registration token shown in the Gitlab CI/CD Runners settings page
* `KUBERNETES_URL` - URL where the cluster can be reached
* `SERVICE_ACCOUNT_BEARER_TOKEN` - The token we've also retrieved from the cluster

Once the instance is up and running, a runner will be activated for the sample repository:

{% asset_img runner.png Registered Runner %}

## Taking it for a spin

Once everything is setup, go back to Gitlab and under CI/CD -> Pipelines, click Run Pipeline. If you've setup everything correctly, you should see the following:

{% asset_img job_successful.png Successful Pipeline %}

Have fun with your shiny new pipeline!
