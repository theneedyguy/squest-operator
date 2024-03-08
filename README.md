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

> [!NOTE] You can also choose a different name for the namespace. Just make sure to also configure the Custom Resource's namespace accordingly

4. Create a manifest with the [Custom Resource](#custom-resource) so that it fits your needs. The parameters are described [here](#configuration). The operator will be installed in the namespace `squest-operator-system`
5. Apply your manifest `kubectl apply -f <path-to-your-manifest.yaml>`

The operator will now attempt to create all needed resources in the `squest` namespace (unless configured differently in the manifest). Finally, you can login to your squest instance via the configured Ingress hostname.

When deploying the manifest to your Cluster it will deploy:

- A Database
- A Redis Cache
- A RebbitMQ Server (Not clustered like the official K8s deployment)
- A migration job to configure your Squest instance before the initial deployment and for upgrades
- The actual Squest deployment.
- A maintenance deployment (As of right now, when upgrading your Squest manifest it will not switch the Ingress to the maintenance deployment during upgrade. This will be added in a later version)
- The celery beat and celery worker deployments
- Persistent Volume Claims, if desired

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
  redis:
    node_selector: |
      kubernetes.io/arch: amd64
    password: toor
  rabbitmq:
    persistence: 
      enabled: true
      storage_class: longhorn
      size: 2Gi
    node_selector: |
      kubernetes.io/arch: amd64
  db:
    persistence: 
      enabled: true
      storage_class: longhorn
      size: 2Gi
    node_selector: |
      kubernetes.io/arch: amd64
    root_password: toor
    password: toor
    user: squest
  squest:
    django:
      ingress:
        enabled: true
        host: "squest.company"
        class: traefik
      image: quay.io/hewlettpackardenterprise/squest:2.5.1
      node_selector: |
        kubernetes.io/arch: amd64
      persistence:
        enabled: true
        storage_class: longhorn
        size: 1Gi
```

## Configuration

This section describes what each parameter of the Custom Resource spec does and how to configure them.

### squest


| Parameter                        | Type    | Description                                         |
| -------------------------------- | ------- | --------------------------------------------------- |
| django.persistence.enabled       | boolean | Enable persistence for the Squest data              |
| django.persistence.storage_class | string  | Name of the storage class for the persistent volume |
| django.persistence.size          | string  | The volume size e.g. 2Gi                            |
| django.node_selector             | map     | Map of strings to select nodes for the deployment   |
| django.image                     | string  | The image to use for the squest deployment          |
| django.ingress.enabled           | boolean | Deploy an Ingress                                   |
| django.ingress.host              | string  | The hostname of your Squest web ui                  |
| django.ingress.class             | string  | The ingress class to use for the ingress            |


### db

| Parameter                 | Type    | Description                                         |
| ------------------------- | ------- | --------------------------------------------------- |
| persistence.enabled       | boolean | Enable persistence for the database                 |
| persistence.storage_class | string  | Name of the storage class for the persistent volume |
| persistence.size          | string  | The volume size e.g. 2Gi                            |
| node_selector             | map     | Map of strings to select nodes for the deployment   |
| root_password             | string  | The database root password                          |
| password                  | string  | The user password                                   |
| user                      | string  | The database user                                   |


### rabbitmq


| Parameter                 | Type    | Description                                         |
| ------------------------- | ------- | --------------------------------------------------- |
| persistence.enabled       | boolean | Enable persistence for rabbitmq                 |
| persistence.storage_class | string  | Name of the storage class for the persistent volume |
| persistence.size          | string  | The volume size e.g. 2Gi                            |
| node_selector             | map     | Map of strings to select nodes for the deployment   |

### redis


| Parameter     | Type   | Description                                       |
| ------------- | ------ | ------------------------------------------------- |
| node_selector | map    | Map of strings to select nodes for the deployment |
| password      | string | The redis cache password                          |


>[!WARNING] You must configure all node_selector parameters right now.
