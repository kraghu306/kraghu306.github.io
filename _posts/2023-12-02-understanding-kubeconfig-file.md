--- 
title: Kubernetes - Understanding the kubeconfig file
categories: [Kubernetes]
comments: false
tags: [kubernetes, kubeconfig]
date: 2023-12-02 00:00:00 +05:30
---

## Pre-read blogs:
1. [Kubernetes Concepts](../kubernetes-concepts/)

Ahoy, curious minds! 🤔

Today, lets continue into the Kubernetes world and explore the kubeconfig file(`~/.kube/config`) - a configuration file which opens doors to the world of Kubernetes. I often think of it as discovering the secrets to unlock and control Kubernetes. 

### Introduction
The Kubernetes configuration file often referred as kubeconfig, which is used to authenticate and interact with Kubernetes. It holds various configuration components such as:
1. Cluster Information - Details of the cluster (includes name, server endpoint and certificate authority(say `.crt` file or `data`))
2. User Information - Details used for authentication such as username, service account, token or client certificate(crt file or data)
3. Context - Its a combination of cluster name, user and namespace, which essentially degining the context to which the user interacts
4. API Versions - Version of the Kubernetes API used by the client

The configuration file is very much crucial for managing multiple Kubernetes clusters or switching between users and namespaces within same cluster or switch to a different cluster altogether. The configuration file is used by the command `kubectl` or client libraries to authenticate and access the Kubernetes resources withing a cluster.

![kubeconfig file](https://blogassets.blob.core.windows.net/images/kubeconfig.png)

### YAML - KubeConfig Sample
```yaml
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: <CA-DATA>
    server: https://<apiServer-IP>:<apiServer-Port>
  name: <clusterName>
contexts:
- context:
    cluster: <clusterName>
    user: <clusterName>
  name: <clusterName>
current-context: <clusterName>
kind: Config
preferences: {}
users:
- name: <clusterName>
  user:
    client-certificate-data: <CLIENT-CRT-DATA>
    client-key-data: <CLIENT-KEY-DATA>
```

### Commands 

#### Generic 

```bash
#Displaying all the configurations from the ~/.kube/config file
kubectl config view

#Displaying the configuration from the custom kube-config file
kubectl config view --kubeconfig=<path-to-kubeconfig>

#Switching between different context using the name
kubectl config use-context <context-name>
```

#### Generating the Configuration
```bash
# Setting the cluster configuration using the Server Endpoint and CA Cert file
kubectl config set-cluster my-cluster --server=https://my-cluster.example.com --certificate-authority=<path-to-ca-file>

# Setting credentials for type User using the Certificate file(.crt) and Client Key file(.key)
kubectl config set-credentials my-user --client-certificate=<path-to-client-cert-file> --client-key=<path-to-client-key-file>

# Setting the context to the cluster, user and namespace
kubectl config set-context my-context --cluster=my-cluster --user=my-user --namespace=default 

#Setting to the Cluster context using the context name
kubectl config use-context my-context
```

#### Using Service Account for authentication
```bash
# Creating a new namespace (Optional)
kubectl create serviceaccount <service-account-name> -n <namespace>

# Creating a new cluster role with permissions and binding to Service account (Optional)
kubectl create clusterrolebinding <role-binding-name> --clusterrole=<cluster-role> --serviceaccount=<namespace>:<service-account-name>

# Fetching the token name for the service account
TOKENNAME=`kubectl -n <namespace> get serviceaccount/<service-account-name> -o jsonpath='{.secrets[0].name}'`

# Obtain the value of Service Account token and decode from base64
TOKEN=`kubectl -n <namespace> get secret $TOKENNAME -o jsonpath='{.data.token}'| base64 --decode`

# Adding the service account(its auth token) as a new user config in kubeconfig file
kubectl config set-credentials <service-account-name> --token=$TOKEN

# Setting the user to current context from the kubeconfig file
kubectl config set-context --current --user=<service-account-name>

# Now we are set to use service account
```

#### Curl approach
```bash
# Fetching the token name for the service account
TOKENNAME=`kubectl -n <namespace> get serviceaccount/<service-account-name> -o jsonpath='{.secrets[0].name}'`

# Obtain the value of Service Account token and decode from base64
TOKEN=`kubectl -n <namespace> get secret $TOKENNAME -o jsonpath='{.data.token}'| base64 --decode`

# We can use Curl to get kubernetes resources, assuming the service account has necessary permissions
$ curl -H "Authorization: Bearer $TOKEN" https://kubernetes/api/v1/pods
```

