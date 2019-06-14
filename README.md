# mig-e2e
End to end tests for OCP 3 to 4 Migration

## Pre-requisites

* OCP3 v3.7 (or higher) and OCP4 clusters, these would be the _source_ and _destination_ clusters used to migrate workloads
* [mig-controller](https://github.com/fusor/mig-controller) must be deployed on _one of the two_ clusters
* Velero must be deployed on both clusters and will be driven by mig-controller see instructions [here](https://github.com/fusor/mig-controller#quick-start)
* S3 temporary migration storage must be provisioned on AWS for mig-controller to consume
* NFS or other compatible volume storage must be provisioned on both clusters for PVC based test

## Mig-controller config variables

A number of parameters need to be set with the correct environment in order to create the CRs for migration purposes. A [sample](https://github.com/fusor/mig-e2e/config/mig_controller.yml.example) file is supplied, please make changes as necessary.  

See below for a decription of these paremeters :

| Parameter | Purpose |
| --- | --- |
| `mig_controller_remote_cluster_url` | Endpoint of remote cluster mig-controller will connect to |
| `mig_controller_aws_access_key` | AWS access key to auth with AWS services |
| `mig_controller_aws_secret_key` | AWS secret access key to auth with AWS services |
| `mig_controller_sa_token` | Base64 encoded SA token used to authenticate with remote cluster |
| `mig_controller_aws_bucket_name` | Name of the S3 bucket to be used for temporary Migration storage |
| `mig_controller_aws_bucket_region` | Region of S3 bucket to be used for temporary Migration storage |

## Other considerations

* You must be _logged in_ to the cluster where mig-controller is running before attempting to run tests
* By default, the source cluster is assumed to be the host for mig-controller

## Running tests

The following example test will deploy and migrate nginx

```bash
ansible-playbook nginx.yml
```

The migration will be tracked to completion, you can also check the status of each _phase of the migration_ using : 

```bash
oc -n mig describe migmigration nginx-mig-1560432273
Name:         nginx-mig-1560432273
Namespace:    mig
Labels:       controller-tools.k8s.io=1.0
Annotations:  touch=ac3a02d7-3249-4d19-9978-53b39cd2f93a
API Version:  migration.openshift.io/v1alpha1
Kind:         MigMigration
Metadata:
  Creation Timestamp:  2019-06-13T13:24:56Z
  Generation:          1
  Resource Version:    16515215
  Self Link:           /apis/migration.openshift.io/v1alpha1/namespaces/mig/migmigrations/nginx-mig-1560432273
  UID:                 a2d9e866-8dde-11e9-8678-0e0ae8a21422
Spec:
  Mig Plan Ref:
    Name:       nginx-migplan-1560432273
    Namespace:  mig
  Stage:        false
Status:
  Completion Timestamp:  2019-06-13T13:28:06Z
  Conditions:
    Category:              Critical
    Last Transition Time:  2019-06-13T13:28:06Z
    Message:               The referenced `migPlanRef` does not have a `Ready` condition.
    Status:                True
    Type:                  PlanNotReady
  Migration Completed:     true
  Phase:                   Completed
  Start Timestamp:         2019-06-13T13:24:57Z
Events:                    <none>
```

_**Note:**_ The migration name will be printed during the ansible playbook run
