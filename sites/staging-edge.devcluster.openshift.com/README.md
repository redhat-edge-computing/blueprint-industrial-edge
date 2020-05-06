## How to deploy staging-edge cluster on AWS

First of all you need to have your Amazon Web Service credentials file located in the following path:

`$HOME/.aws/credentials`

This file looks like this:

```
[default]
aws_access_key_id = xxxx
aws_secret_access_key = xxxx
```

The OpenShift installer binary will read that file if aws is set as a platform. From the path where the `knictl` binary is located, and in order to pull our staging-edge site and its requirements, please execute:

`knictl fetch_requirements github.com/redhat-edge-computing/blueprint-industrial-edge/sites/staging-edge.devcluster.openshift.com/`

This command will download the site blueprint definition, and all its requirements (oc, openshift-install, kustomize, etc) to the `$HOME/.kni/`. Every site will have a separate directory within that location. The next step involves the actual rendering of the manifests (site + profile + base) into one set of manifests via kustomize that we can pass to the openshift-install binary.

`knictl prepare_manifests staging-edge.devcluster.openshift.com`

If everything goes well, the command will get out some instructions to deploy the cluster. It's basically asking you to run `openshift-install` binary pointing to where the final manifests created by `knictl` are:

`$HOME/.kni/staging-edge.devcluster.openshift.com/requirements/openshift-install create cluster --dir=$HOME/.kni/staging-edge.gcp.devcluster.openshift.com/final_manifests --log-level debug`

Wait until the deployment is completed, and you will information about console endpoint, kubeadmin password and kubeconfig path. 

If you have manifests that you want to deploy as Day 2 operations located in any of the 02_cluster-addons or 03_services directories, you can deploy them running the following command:

`knictl apply_workloads staging-edge.devcluster.openshift.com`

This is basically running kustomize to build and render all the manifests enabling alpha plugins, and apply them via oc/kubectl.
