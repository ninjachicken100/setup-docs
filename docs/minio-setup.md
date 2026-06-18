# Overview

This guide is for Dev Team to deploy MinIO S3 Bucket on **OpenShift Local** for development testing purposes.


## Install Guide
For files that are applied, please refer to the [Github Repo](https://github.com/ninjachicken100/setup-docs)


### MinIO Operator

Pull from [MinIO](https://github.com/minio/minio/tree/master/helm/minio) Github Repo 


**Create a new namespace**

```
oc new-project minio
```

**Configure the values-override.yaml**

```
# -- Number of pod replicas
replicas: 1
mode: standalone

# -- Pod security context configuration. 
# Run 'oc get events' to debug which UID is permissible
securityContext:
  # -- User ID to run the container process
  runAsUser: {}
  # -- Primary group ID for the container process
  runAsGroup: {}
  # -- File system group ID for volume permissions
  fsGroup: {}

# -- Persistence configuration for object storage
persistence:
  # -- Size of the PVC to create (if set and no existingClaim, a new PVC is created)
  size: 50Gi # adjust bucket space as required

# -- Resource configuration
resources:
  requests:
    # -- CPU memory configuration
    memory: 8Gi
```

**Run Helm Install**

```
helm install minio ./ -f values-override.yaml --set rootUser=<username>,rootPassword=<password>
```
Check that your MinIO pod is running and that the route is accessible.

### Troubleshoot
If there are no routes, list out all the services within the minio namespace

```
oc get svc -n minio
```
Look out for the minio-console service and expose it

```
oc expose svc <svc-name>
```