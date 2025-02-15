# NOTE

**This is alpha software, use it on your own risk!**

# kubepost Operator

The kubepost Operator manages various Postgres objects via standard Kubernetes CRDs. It requires [Metacontroller]
(https://github.com/metacontroller/metacontroller) to be installed within the cluster.

## Features

- Role lifecycle management
- Database lifecycle management
- Database extension lifecycle management
- Permission lifecycle management

## PostgreSQL

At the moment only Postgres 13 is supported, backwards compatibility might be added at a later stage.

## Installation

### Metacontroller

kubepost uses Metacontroller under the hood, so this component hast to be installed within the cluster. You can view
detailed instructions for the installation
process [here](https://metacontroller.github.io/metacontroller/guide/install.html).

Create a new kustomization file with the following content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: metacontroller

resources:
  - github.com/metacontroller/metacontroller/manifests//production
```

And apply it to the desired cluster:

```bash
kustomize build . | kubectl apply -f -
```

### kubepost

After you have installed metacontroller you have to deploy kubepost. Create a new kustomization file with the following
content:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: metacontroller

resources:
  - github.com/orbatschow/kubepost//manifests
```

And apply it to the desired cluster:

```bash
kustomize build . | kubectl apply -f -
```

## Usage

### Instance

The instance is used by other CRDs to connect to the desired database instance. It allows
a clear segregation between roles, databases and the instance itself. To connect to the databse
it uses a secret that should be available within the Kubernetes cluster.

```yaml
apiVersion: kubepost.io/v1alpha1
kind: Instance
metadata:
  name: kubepost # this is the instanceRef, used by other CRDs
spec:
  host: localhost
  port: 5432
  database: postgres
  secretRef:
    name: kubepost-instance-credentials
    namespace: default
    userKey: username
    passwordKey: password
```

The corresponding secret might look like this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: kubepost-instance-credentials
data:
  username: cG9zdGdyZXM=
  password: cm9vdA==

```

### Database


This database uses the previously mentioned instance CRD to connect to the database
instance and creates a database with the name `kubepost`. After the database is created
the extension `pg_stat_statements` will be installed.

```yaml
apiVersion: kubepost.io/v1alpha1
kind: Database
metadata:
  name: kubepost
  namespace: default
spec:
  databaseName: kubepost
  preventDeletion: false
  instanceRef:
    name: kubepost
    namespace: default
  extensions:
    - pg_stat_statements
```

### Role

This role uses the previously mentioned instance CRD to connect to the database
instance and creates a role with the name `kubepost`. It then grants this role
`ALL PRIVILEGES` on the database `kubepost`. The grant section is optional.

```yaml
apiVersion: kubepost.io/v1alpha1
kind: Role
metadata:
  name: kubepost
  namespace: default
spec:
  roleName: kubepost
  preventDeletion: false
  instanceRef:
    name: kubepost
    namespace: default
  grant:
    database: kubepost
    objectType: database
    privileges:
      - ALL PRIVILEGES

```
