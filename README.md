# Kubernetes RBAC

# Introduction

This example uses the users group by generating a certificate from the cluster's CA.
For configuring a proper authentication/authorization mechanism use an external OIDC provider. (ex: [Amazon](https://www.eksworkshop.com/beginner/091_iam-groups), Azure, GCP, Keycloak, dex..)


# Steps

## Create a cluster 

If you don't have a cloud cluster, create your own in your working computer:
```sh
kind create cluster --name rbac
```

## Configure KUBECONFIG (skip this step if you are using an external OIDC provider)

### Retrieve the cluster's CA certificate

To grab the cluster's CA we need to connect to the node and download the certificate.

```sh
$ docker ps

CONTAINER ID   IMAGE                  COMMAND                  CREATED             STATUS                 PORTS                                                                 NAMES
9592e9e8f2dc   kindest/node:v1.21.1   "/usr/local/bin/entr…"   About an hour ago   Up About an hour       0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp, 127.0.0.1:40045->6443/tcp   rbac-control-plane

docker exec -it rbac-control-plane cat /etc/kubernetes/pki/ca.crt > ca.crt
docker exec -it rbac-control-plane cat /etc/kubernetes/pki/ca.key > ca.key
```

### Generate the certificate for the user

- Create private key
```sh
openssl genrsa -out dev1.key 2048
```

- Create Certificate Signing Request (CSR) with user "dev1" and group "devs".
```sh
openssl req -new -key dev1.key -out dev1.csr -subj "/CN=dev/O=devs"
```

- Create Certificate from CSR using the cluster's CA
```sh
openssl x509 -req -in dev1.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out dev1.crt -days 365
```

### Configure KUBECONFIG with the new user

- Add the new cluster (Make sure its not already there, if it is skip this step)
```sh
k config set-cluster kind-rbac --certificate-authority=ca.crt --embed-certs=true --server=https://127.0.0.1:40045
```

- Add the new user credentials
```sh
k config set-credentials dev1 --client-certificate=dev1.crt --client-key=dev1.key --embed-certs=true
```

- Add the new context
```sh
k config set-context kind-rbac-dev1 --cluster=kind-rbac --user=dev1
```

- Check if the new user is created
```sh
k config get-contexts
CURRENT   NAME             CLUSTER     AUTHINFO    NAMESPACE
*         kind-rbac        kind-rbac   kind-rbac   
          kind-rbac-dev1   kind-rbac   dev1        


k config use-context kind-rbac-dev1
```

## Assign permissions to the user's group

If you tried to issue a kubectl command with the `dev1` user's context, you see the following error:

```
k get pods
Error from server (Forbidden): pods is forbidden: User "dev" cannot list resource "pods" in API group "" in the namespace "default"
```

When a user is created the default behavior is deny all access to the cluster.
To give permissions we need to assign the user or group roles.

### Creating a Role for the group

There is already 3 default cluster roles created:
- view: Only read-only access to k8s objects
- edit: Read-Write access except roles management.
- cluster-admin: The default used for the cluster admin user.

If you need a more tailored role copy from this existing ones and change them or take advantage of [aggregate roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)

### Assign a role to the group

As you remember, our user 'dev1' belongs to group 'dev' when we created the certificate.
Usually the user's group comes from the oidc provider's token but since we are using certificate based authentication then its the Organization Name from the certificate that's used.

Also, if you want the user to have only permissions in a particular namespace then you should use `rolebinding` to that particular namespace.

Example for a namespaced role:
```sh
k create rolebinding devs-binding -n my-namespace --clusterrole=edit --group=devs
```

Example for a cluster wide role:
```sh
k create clusterrolebinding devs-cluster-binding -n my-namespace --clusterrole=edit --group=devs
```

If you want to generate a yaml from the command above just add `-oyaml --dry-run` at the end of the command.

# References

[CNCF Webniar: RBAC policies in Kubernetes](https://www.youtube.com/watch?v=CnHTCTP8d48)
[Agreggate multiple Roles](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)