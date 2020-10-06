# How to deploy a site cluster on AWS

This is a two step process:

1. First the user has to define its site cluster and push it to its git reporistory.
2. Then, the user shoud use `knictl` to render all the manifests, and run the OCP instaler.

## Define your site cluster

### Create site folder

First of all, you have to clone this repo, and use one of the site clusters as a baseline:

For example, if you want to deploy a new cluster (site) on AWS, you can copy the site `staging-edge.devcluster.openshift.com`.

For the purpose of this example, we are assuming the new site is called: `staging-edge.devcluster.openshift.com` and the profile `production.aws`.

```bash
export SITE_NAME='staging-edge.devcluster.openshift.com'
export PROFILE_NAME='production.aws'

cd blueprint-industrial-edge/sites
cp -a staging-edge.devcluster.openshift.com/ "$SITE_NAME/"
```

### Edit profile requirements

The user should know which exact version of OCP wants to deploy, and should edit the `requirements.yaml` file
accordingly, to download the right client tools versions.

Also, the user should know which version of kubernetes is using the exact version of OCP.

For example, for OCP 4.4, k8s 1.17 is used. The user should always check [OCP release notes](https://docs.openshift.com/container-platform/4.4/release_notes/ocp-4-4-release-notes.html#ocp-4-4-about-this-release) before.

```bash
cd blueprint-industrial-edge/profiles/
vi "$PROFILE_NAME/requirements.yaml"
```

#### Edit 00_install

```bash
cd blueprint-industrial-edge/sites/$SITE_NAME/00_install-config
```

+ **kustomization.yaml** -> Change git url.
+ **install-config.patch.yaml** -> Change baseDomain, and alterantively other fields, such as the aws zone for example.
+ **install-config.name.patch.yaml** -> Change the cluster-name.

#### Edit 02_cluster_addons

If registering the site againts a RHACM hub, then:

```bash
cd blueprint-industrial-edge/sites/$SITE_NAME/02_cluster_addons/00_acm_registration
```

 + **acm-name-config.patch.yaml** -> Change clusterName and clusterNamespace, use the same value.

##### Git push

Push the changes to your repo, the url go the git repo should match the git url you have specified in previous steps:

```bash
cd blueprint-industrial-edge
git add .
git commit -m "Adds my new blueprint site"
git push <my-remote> master
```

## Deploy your site cluster

### AWS credentials

First of all you need to have your Amazon Web Service credentials file located in the following path:

`$HOME/.aws/credentials`

This file looks like this:

```
[default]
aws_access_key_id = xxxx
aws_secret_access_key = xxxx
```

The OpenShift installer binary will read that file if aws is set as a platform.

### Prepare .kni folder

Your `.kni/` folder should contain the following files, otherwise either your deployment or Day 2 workloads will fail to be deployed.

```bash
tree .kni/
.
├── dockerconfig.json
├── id_rsa
├── id_rsa.pub
├── kubeconfighub.json
├── pull-secret.json
```

+ **dockerconfig.json:**  It is a valid pull secret to pull RHACM images on the registered cluster. Only needed if you want your OCP cluster to be autoregisters against a RHACM Hub cluster.

It is basically a base64 encoded pull secret. To generate it, just execute:

```bash
cd ~/.kni/
cat pull-secret.json | base64 -w0 > ~/.kni/dockerconfig.json
```

+ **kubeconfighub.json:** It is the the kubeconfig of the RHACM hub cluster, base64 encoded. It is used by the RHACM Endpoint pod to register itself against the RHACM.

To generate it, just execute:

```bash
cat rhacm-hub-kubeconfig | base64 -w0 > ~/.kni/kubeconfighub.json
```

### Environment vars & aliases

Define the following var and aliases, according to your needs.

```bash
export SITE_NAME='staging-edge.devcluster.openshift.com'
export GIT_REPO='github.com/redhat-edge-computing/blueprint-industrial-edge'
alias openshift-install="$HOME/.kni/$SITE_NAME/requirements/openshift-install"
```

### kncitl: Preparation steps

From the path where the `knictl` binary is located, and in order to pull our site and its requirements, please execute:

```bash
knictl fetch_requirements "$GIT_REPO/sites/$SITE_NAME/"
```

This command will download the site blueprint definition, and all its requirements (oc, openshift-install, kustomize, etc) to the `$HOME/.kni/`. Every site will have a separate directory within that location.

The next step involves the actual rendering of the manifests (site + profile + base) into one set of manifests via kustomize that we can pass to the openshift-install binary.

```bash
knictl prepare_manifests "$SITE_NAME"
```

If everything goes well, the command will get out some instructions to deploy the cluster. It's basically asking you to run `openshift-install` binary pointing to where the final manifests created by `knictl` are.

### Deploy OpenShift

Just execute the following command:

```bash
openshift-install create cluster --dir="$HOME/.kni/$SITE_NAME/final_manifests" --log-level debug
```

Wait until the deployment is completed, and you will information about console endpoint, kubeadmin password and kubeconfig path.

### knictl: deploy Day 2 workloads

If you have manifests that you want to deploy as Day 2 operations located in any of the `02_cluster-addons` or `03_services directories`, you can deploy them running the following command:

```bash
knictl apply_workloads "$SITE_NAME"
```

This is basically running kustomize to build and render all the manifests enabling alpha plugins, and apply them via oc/kubectl.

**NOTE:**: If for some reasons the previous command fails, you can check the kustomize rendered manifests under `/tmp`,
or under `~/.kni/tmp` if using a containerized version of knictl.

### Destroy OpenShift cluster

To destroy your site cluster:

```bash
openshift-install destroy cluster --dir="$HOME/.kni/$SITE_NAME/final_manifests" --log-level debug
```