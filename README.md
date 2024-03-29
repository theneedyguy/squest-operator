# Squest Operator

Deploy [Squest](https://github.com/HewlettPackard/squest) in a controlled way using an [Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/). 

> [!NOTE]
> This Operator is not made by Hewlett Packard but is a community project. 


## Why an Operator

HP offers an Ansible Playbook to help users deploy HP's Squest on their K8s Clusters. The play by default installs a lot of other tools not necessarily needed to run a Squest instance like Prometheus or CertManager. Also, the whole lifecycle of the Squest instance requires you to manually run Ansible playbooks. With this operator, the upgrade is as simple as changing the image tag in the custom resource `Squest`, triggering a redeployment and migration.

## Getting started

### Installation

1. To install the Operator, clone the repo and change directory into the cloned directory.
2. Run `make deploy` to install all required resources and the operator into your cluster. This requires you to have an existing K8s context on your system.
3. Create new namespace for your Squest deployment to live in: `kubectl create ns squest` 

> [!NOTE] 
> You can also choose a different name for the namespace. Just make sure to also configure the Custom Resource's namespace accordingly

4. Create a manifest with the [Custom Resource](#custom-resource) so that it fits your needs. The parameters are described [here](#configuration). The operator will be installed in the namespace `squest-operator-system`
5. Apply your manifest `kubectl apply -f <path-to-your-manifest.yaml>`

The operator will now attempt to create all needed resources in the `squest` namespace (unless configured differently in the manifest). Finally, you can login to your squest instance via the configured Ingress hostname.

When deploying the manifest to your Cluster it will deploy:

- A Database
- A Redis Cache
- A RabbitMQ Server (Not clustered like the official K8s deployment)
- A migration job to configure your Squest instance before the initial deployment and for upgrades
- The actual Squest deployment.
- A maintenance deployment (During initial deployment and during upgrades, the ingress - should you have enabled it in the custom resource - will switch over to this deployment's service to display the maintenance page.)
- The celery beat and celery worker deployments
- Persistent Volume Claims, if desired

### Upgrade 

To upgrade to a newer version of Squest, simply change the image value in the [Custom Resource](#custom-resource) called `django_image` to a new version and apply the manifest. The operator will automatically create a new migration job and redeploy Squest with the new version.

### Uninstall

To remove the operator and the Squest resource and all the created dependencies run `make undeploy`.

## Custom Resource

This custom resource defines your Squest environment and configures all the resources. Copy this sample and configure it to your liking.

```yaml
apiVersion: web.hewlettpackard.com/v1alpha1
kind: Squest
metadata:
  labels:
    app.kubernetes.io/name: squest
    app.kubernetes.io/instance: squest
    app.kubernetes.io/part-of: squest-operator
    app.kubernetes.io/managed-by: kustomize
    app.kubernetes.io/created-by: squest-operator
  name: squest
  namespace: squest
spec:
  node_selector: ''
  redis_node_selector: ''
  rabbitmq_node_selector: ''
  db_node_selector: ''
  django_node_selector: |
    kubernetes.io/arch: amd64
  maintenance_node_selector: ''

  storage_class: default

  db_persistence_enabled: true
  db_persistence_storage: 2Gi
  db_user: squest
  db_password: toor
  db_root_password: toor

  rabbitmq_persistence_enabled: true
  rabbitmq_persistence_storage: 2Gi

  django_persistence_enabled: true
  django_persistence_storage: 1Gi
  django_ingress_enabled: false
  django_ingress_class: traefik
  django_ingress_hostname: squest.0x01.host
  django_image: quay.io/hewlettpackardenterprise/squest:2.5.1

  redis_password: toor

```

## Configuration

This section describes what each parameter of the Custom Resource spec does and how to configure them.

### global parameters

| Parameter     | Type   | Description                                                 | Default |
| ------------- | ------ | ----------------------------------------------------------- | ------- |
| storage_class | string | Storage Class for all Persistent Volumes                    | default |
| node_selector | string | Multiline string of nodeSelectors to apply to all resources | ''      |

### squest

| Parameter                    | Type    | Description                                                                                  | Default                                        |
| ---------------------------- | ------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| django_persistence_enabled   | boolean | Enable persistence for the Squest data                                                       | true                                           |
| django_persistence_storage   | string  | The volume size e.g. 2Gi                                                                     | 1Gi                                            |
| django_node_selector         | string  | Multiline string to select nodes for the deployment                                          | ''                                             |
| django_image                 | string  | The image to use for the squest deployment                                                   | quay.io/hewlettpackardenterprise/squest:latest |
| django_ingress_enabled       | boolean | Deploy an Ingress                                                                            | false                                          |
| django_ingress_hostname      | string  | The hostname of your Squest web ui                                                           | squest.company                                 |
| django_ingress_class         | string  | The ingress class to use for the ingress                                                     | traefik                                        |
| django_resource_requirements | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br> 	&nbsp; 	&nbsp;memory: 512Mi   |

### db

| Parameter                | Type    | Description                                                                                  | Default                  |
| ------------------------ | ------- | -------------------------------------------------------------------------------------------- | ------------------------ |
| db_persistence_enabled   | boolean | Enable persistence for the database                                                          | true                     |
| db_persistence_storage   | string  | The volume size e.g. 2Gi                                                                     | 2Gi                      |
| db_node_selector         | string  | Multiline string to select nodes for the deployment                                          | ''                       |
| db_root_password         | string  | The database root password                                                                   | toor                     |
| db_password              | string  | The user password                                                                            | toor                     |
| db_user                  | string  | The database user                                                                            | squest                   |
| db_resource_requirements | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br> 	&nbsp; 	&nbsp;memory: 512Mi |

### rabbitmq

| Parameter                      | Type    | Description                                                                                  | Default                    |
| ------------------------------ | ------- | -------------------------------------------------------------------------------------------- | -------------------------- |
| rabbitmq_persistence_enabled   | boolean | Enable persistence for rabbitmq                                                              | true                       |
| rabbitmq_persistence_storage   | string  | The volume size e.g. 2Gi                                                                     | 2Gi                        |
| rabbitmq_node_selector         | string  | Multiline string to select nodes for the deployment                                          | ''                         |
| rabbitmq_resource_requirements | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br> 	&nbsp; 	&nbsp;  memory: 256Mi |

### redis

| Parameter                   | Type   | Description                                                                                  | Default                   |
| --------------------------- | ------ | -------------------------------------------------------------------------------------------- | ------------------------- |
| redis_password              | string | The redis cache password                                                                     | toor                      |
| redis_node_selector         | string | Multiline string to select nodes for the deployment                                          | ''                        |
| redis_resource_requirements | map    | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br> 	&nbsp; 	&nbsp;  memory: 50Mi |

### maintenance

| Parameter                         | Type | Description                                                                                  | Default                   |
| --------------------------------- | ---- | -------------------------------------------------------------------------------------------- | ------------------------- |
| maintenance_node_selector         | string  | Multiline string to select nodes for the deployment                                       | ''                        |
| maintenance_resource_requirements | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br> 	&nbsp; 	&nbsp;  memory: 16Mi |