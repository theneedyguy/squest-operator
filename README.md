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
- A RebbitMQ Server (Not clustered like the official K8s deployment)
- A migration job to configure your Squest instance before the initial deployment and for upgrades
- The actual Squest deployment.
- A maintenance deployment (As of right now, when upgrading your Squest manifest it will not switch the Ingress to the maintenance deployment during upgrade. This will be added in a later version)
- The celery beat and celery worker deployments
- Persistent Volume Claims, if desired

### Upgrade 

To upgrade to a newer version of Squest, simply change the image value in the [Custom Resource](#custom-resource) under `squest.django.image` to a new version and apply the manifest. The operator will automatically create a new migration job and redeploy Squest with the new version.

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
    resources:
      limits:
        memory: 50Mi
  rabbitmq:
    persistence: 
      enabled: true
      storage_class: longhorn
      size: 2Gi
    node_selector: |
      kubernetes.io/arch: amd64
    resources:
      limits:
        memory: 256Mi
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
    resources:
      limits:
        memory: 512Mi
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
      resources:
        limits:
          memory: 512Mi
  maintenance:
    node_selector: |
      kubernetes.io/arch: amd64
    resources:
      limits:
        memory: 16Mi
```

## Configuration

This section describes what each parameter of the Custom Resource spec does and how to configure them.

### squest


| Parameter                        | Type    | Description                                                                                  | Default                                        |
| -------------------------------- | ------- | -------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| django.persistence.enabled       | boolean | Enable persistence for the Squest data                                                       | true                                           |
| django.persistence.storage_class | string  | Name of the storage class for the persistent volume                                          | default                                        |
| django.persistence.size          | string  | The volume size e.g. 2Gi                                                                     | 1Gi                                            |
| django.node_selector             | string  | Multiline string to select nodes for the deployment                                          | ''                                             |
| django.image                     | string  | The image to use for the squest deployment                                                   | quay.io/hewlettpackardenterprise/squest:latest |
| django.ingress.enabled           | boolean | Deploy an Ingress                                                                            | false                                          |
| django.ingress.host              | string  | The hostname of your Squest web ui                                                           | squest.company                                 |
| django.ingress.class             | string  | The ingress class to use for the ingress                                                     | traefik                                        |
| django.resources      | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br>&nbsp;&nbsp;memory: 512Mi                     |

### db

| Parameter                 | Type    | Description                                                                                  | Default                  |
| ------------------------- | ------- | -------------------------------------------------------------------------------------------- | ------------------------ |
| persistence.enabled       | boolean | Enable persistence for the database                                                          | true                     |
| persistence.storage_class | string  | Name of the storage class for the persistent volume                                          | default                  |
| persistence.size          | string  | The volume size e.g. 2Gi                                                                     | 2Gi                      |
| node_selector             | string  | Multiline string to select nodes for the deployment                                          | ''                       |
| root_password             | string  | The database root password                                                                   | toor                     |
| password                  | string  | The user password                                                                            | toor                     |
| user                      | string  | The database user                                                                            | squest                   |
| resources      | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br>&nbsp;&nbsp;memory: 512Mi |

### rabbitmq


| Parameter                 | Type    | Description                                                                                  | Default                    |
| ------------------------- | ------- | -------------------------------------------------------------------------------------------- | -------------------------- |
| persistence.enabled       | boolean | Enable persistence for rabbitmq                                                              | true                       |
| persistence.storage_class | string  | Name of the storage class for the persistent volume                                          | default                    |
| persistence.size          | string  | The volume size e.g. 2Gi                                                                     | 2Gi                        |
| node_selector             | string  | Multiline string to select nodes for the deployment                                          | ''                         |
| resources      | map     | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br>&nbsp;&nbsp;memory: 256Mi |
### redis


| Parameter            | Type   | Description                                                                                  | Default                   |
| -------------------- | ------ | -------------------------------------------------------------------------------------------- | ------------------------- |
| node_selector        | string | Multiline string to select nodes for the deployment                                          | ''                        |
| password             | string | The redis cache password                                                                     | toor                      |
| resources | map    | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br>&nbsp;&nbsp;memory: 50Mi |

### maintenance

| Parameter            | Type   | Description                                                                                  | Default                   |
| -------------------- | ------ | -------------------------------------------------------------------------------------------- | ------------------------- |
| node_selector        | string | Multiline string to select nodes for the deployment                                          | ''                        |
| resources | map    | Define the resource limits and requests according to the official Kubernetes deployment spec | limits:<br>&nbsp;&nbsp;memory: 16Mi |