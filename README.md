
# OCP MIG CI

## Purpose

This repo contains ansible and CI related assets used for the OCP 3 to 4 migration project, the primary use case for these tools is to integrate with Jenkins and allow the creation of all necessary CI workflows. It will consist of several ansible playbooks, which will prepare a environment for migration purposes. This involves provisioning customer-like cluster deployments with [OA installer](https://github.com/openshift/openshift-ansible), and setup of the migration tools from [mig-controller](https://github.com/fusor/mig-controller).

## Pipelines 

Jenkins pipelines are used to provide the logic necessary to orchestrate the build and execution of CI workflows. Each pipeline job can be parameterized to customize the behavior for the intended workflow, they are also in charge of providing notifications for each build, below are the supplied  [mig CI pipelines](https://github.com/fusor/mig-ci/tree/master/pipeline) with a brief description:

| Pipeline | Purpose |
| -------- | ------- |
| `ocp3-agnosticd-base` | Deploys OCP3 using agnosticd, performs cluster sanity checks, multi-node cluster support |
| `ocp3-oa-base` | Deploys OCP3 using openshift ansible, performs cluster sanity checks, all-in-one cluster |
| `ocp3-origin3-dev-base` | Deploys OCP3 using [origin3-dev](https://github.com/fusor/origin3-dev.git), all-in-one cluster |
| `ocp4-base` | Deploys OCP4 and performs cluster sanity checks |
| `parallel-base` | Deploys OCP3, OCP4, NFS server in parallel, installs cluster application migration tools and executes e2e migration tests|

### Pipeline CI jobs logic

The use of *trigger pipeline jobs* which are parameterized is key to the structure of the CI workflows, trigger jobs can for instance watch repositories for activity and execute *base pipelines* to run a CI workflow. A good example are trigger pipelines watching the [mig-controller repo](https://github.com/fusor/mig-controller) for changes and executing the [parallel base pipeline](https://github.com/fusor/mig-ci/blob/master/pipeline/parallel-base.groovy) with [e2e tests](https://github.com/fusor/mig-e2e)

### Pipeline CI job parameters

These are some of the parameters allowing the customizations of mig CI jobs :

| Parameter | Purpose | Notes |
| ----- | ---------- | ---------- |
| `AWS_REGION` | AWS region for clusters to deploy | |
| `OCP3_VERSION`| OCP3 version to deploy | Default is v3.11 |
| `OCP4_VERSION`| OCP4 version to deploy | Default is v4.1 |
| `NODE_COUNT` | Number of compute nodes to create | Will be the same for source and target clusters |
| `MASTER_COUNT` | Number of master nodes to create | Will be the same for source and target clusters |
| `CLUSTER_NAME` | Name of the cluster to deploy | The final deployment will use the following convention: `${CLUSTER_NAME}-v3-${BUILD_NUMBER}-${OCP3_VERSION}`. In AWS you can this value on instance tags GUID label|
| `EC2_KEY` | Name of ssh public and private key, which will be imported in OCP3 cluster and allow ssh access | By default is `ci`, but for manual run is recomended `libra`|
| `DEPLOYMENT_TYPE` | OCP3 deployment type | Could be `agnosticd`, `OA` or `cluster_up`|
| `MIG_CONTROLLER_REPO` | source repository for mig-controller to test | |
| `MIG_CONTROLLER_BRANCH` | source branch for mig-controller to test | |
| `SUB_USER` | RH subscription username | Only used in OA deployments to access OCP bits |
| `SUB_PASS` | RH subscription password | Only used in OA deployments to access OCP bits |
| `CLEAN_WORKSPACE` | Clean Jenkins workspace after build | Boolean |
| `EC2_TERMINATE_INSTANCES` | Terminate all instances on EC2 after build | Boolean, clean up of clusters and other CI EC2 related instances |

_**Note:**_ **For a full list of all possible parameters please inspect each pipeline script**

### Migration controller e2e tests

The migration controller e2e tests are supplied in the [mig-e2e repo](https://github.com/fusor/mig-e2e), the tests are based on [mig controller sample scenarios](https://github.com/fusor/mig-controller/tree/master/docs/scenarios) and are executed during the last stage of CI jobs.

### Debug and cleanup of CI jobs

The `EC2_TERMINATE_INSTANCES` and `CLEAN_WORKSPACE` boolean parameters can be used to avoid the termination of clusters in case you want to debug the migration after the run: 

1) Ssh to jenkins host
2) Go to `/var/lib/jenkins/workspace`, and enter the `parallel-mig-ci-${BUILD_NUMBER}` dir. The `kubeconfigs` directory will contain `KUBECONFIG` files with opened sessions for both source and target cluster. You can utilize them with `export KUBECONFIG=$(pwd)/kubeconfigs/ocp-v3.11-kubeconfig`.
3) After finishing those steps you can remove deployments manually by executing `./destroy_env.sh`.

## External NFS server setup on AWS

In order to demonstrate the migration of PV resources, you can deploy an external NFS server on AWS. The server will have a public IP, and could be pointed from both locations - source OCP3 and target OCP4 based clusters.

Instance creation and NFS server setup is handled by the following command:
- `ansible-playbook nfs_server_deploy.yml`

From that moment you have the NFS server running. The next step is to login into the cluster, where you need to provision the PV resources. Then you are ready to create the PV resources on the cluster by running:
- `ansible-playbook nfs_provision_pvs.yml`
This task could be executed repetitively, for every cluster you need to bond with the NFS server.

When you are finished, just run the playbook to destroy NFS AWS instance:
- `ansible-playbook nfs_server_destroy.yml`

Functionality of this tool is tested with both [all-in-one](https://github.com/fusor/mig-ci#ocp3-all-in-one-deployment-on-aws) AWS deployment, and [origin3-dev](https://github.com/fusor/origin3-dev/).

## OCP3 agnosticd multinode in AWS

This type of deployment is used in [parallel-base](https://github.com/fusor/mig-ci/blob/master/pipeline/parallel-base.groovy) pipeline, and is used for creation of multinode cluster. To setup a similar environment outside of CI, please refer to the [official](https://github.com/redhat-cop/agnosticd) doc.

## OCP3 OA all-in-one deployment on AWS

In order to execute an OCP3 all-in-one deployment, you should specify several environment variables.

Customer environment is expected to be based on RHEL distributions. By default the RHEL 7 does not have an access to the [official OpenShift bits](https://docs.openshift.com/enterprise/3.0/install_config/install/prerequisites.html#software-prerequisites) for the OA deployment, so you need to setup a valid account with those subscriptions by following variables:

Task specific:

- `OCP3_VERSION` - verison of cluster to be provisioned. Should be specified as 'v3\.[0-9]+'. If not set, will be used 'v3.11'.

- `WORKSPACE` - from [other variables](https://github.com/fusor/mig-ci#list-of-other-environment-variables). You should create a directory `$WORKSPACE/keys` and place there your AWS ssh private key, which will be supplied to the newly created instance. The name of the key is captured from `$EC2_KEY`.
  - The result ssh private key file will should be discoverable in `${WORKSPACE}/keys/${EC2_KEY}.pem`. You can use it after that, to access the instance via ssh.

### Deploying OA cluster outside of CI :

- Setup all of the [environment variables](https://github.com/fusor/mig-ci#list-of-other-environment-variables), including specific for this task, mentioned in the previous paragraph.

- To deploy an OA `ansible-playbook deploy_ocp3_cluster.yml -e prefix=name_for_instance`. You can specify deployment type with `-e oa_deployment_type=openshift-enterprise/origin`. Default is the enterprise one. When prefix is not specified, you `ansible_user` variable will be used.

If you want to select the downstream version of openshift, you can add `-e oa_deployment_type=origin` tag to previous step.

- To destroy the instance and all attached resources `ansible-playbook destroy_ocp3_cluster.yml -e prefix=name_for_instance`.

Deployment usually takes around 40-80 minutes to complete. Logs are written in `$WORKSPACE/.install.log` file upon completion.

In case that you don't want to specify environment variables, every `deploy*` and `destroy*` playbook could be run with config file located in `/config/adhoc_vars.yml`.

Example:

- `ansible-playbook deploy_ocp3_cluster.yml -e @config/adhoc_vars.yml`
- `ansible-playbook deploy_ocp4_cluster.yml -e @config/adhoc_vars.yml`
- `ansible-playbook destroy_ocp3_cluster.yml -e @config/adhoc_vars.yml`
- `ansible-playbook destroy_ocp4_cluster.yml -e @config/adhoc_vars.yml`

### List of other environment variables:

- `SUB_USER` - redhat subscription username for account, which have access to the openshift bits. Allows to setup an `enterprise` and `origin` OA deployment.

- `SUB_PASS` - password for the redhat subscriprion account.

- `AWS_REGION` - region, in which all resources will be created.

- `AWS_ACCESS_KEY_ID` - AWS access key id, which is used to access your AWS environment.

- `AWS_SECRET_ACCESS_KEY` - secret, used for authentication.

- `WORKSPACE` - Should specify a placement of workspace, which contains the `keys` directory, with all the ssh private and public keys used during the run are located. The name of the key is specified by the `EC2_KEY` variable. If was not specified, will be used the default value - `../`.

- `EC2_KEY` - name of the ssh private key, which will be passed to the instance, and allow you to access the instance via ssh in future. Set to `ci` by default. The key should be discoverable with `${WORKSPACE}/keys/${EC2_KEY}.pem`.

- `EC2_REGION` - region, where all resources will be created.

- `CLUSTER_NAME` - this variable is used in multiple location. In this scenario it's perpouse, is to specify prefix for a newly created AWS EC2 instance. When it is not specified, your ansible username will be used. All EC2 instances are named by the following convention: `$CLUSTER_NAME-<instance role>-3.(7-11)`.

- `KUBECONFIG` - if this envrironment variable is set, then the `oc` binary will use the configuration file from there to perform any operations on cluster. By default the `~/.kube/config` location is used. https://docs.openshift.com/container-platform/3.11/cli_reference/manage_cli_profiles.html

- - - -
