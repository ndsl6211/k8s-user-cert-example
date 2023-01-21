# Client Cert in K8s

## Description
To have a new user to be able to authenticate Kubernetes API, we must create a
user with the mechanism of **Certificate Signing Request**

## Prerequisite
1. Have `openssl` CLI installed
1. Have `kubectl` CLI installed
1. Have a Kubernetes cluster

## Steps
1. generate a private key for the user
    ```bash
    openssl genrsa -out user.key 2048
    ```
    - `genrsa` subcommand would help us to generate a rsa key
    - `-out user.key` specifies the output key file
    - `2048` specifies the size of the private key to generate in bits


1. generate a CSR (certificate signing request) for that user
    ```bash
    openssl req -new -key user.key -out user.csr
    ```
    - `req` subcommand would creates and processes certificate requests in
    PKCS#10 format
    - `-new` specify to **create a new file**
    - `-key user.key` specify the key to use to generate CSR
    - `-out user.csr` specify the output csr file

1. create a CSR to K8s cluster (k8s yaml file)
    ```
    cat <<EOF | kubectl apply -f -
    apiVersion: certificates.k8s.io/v1
    kind: CertificateSigningRequest
    metadata:
        name: user-csr
    spec:
        request: $(cat user.csr | base64 | tr -d '\n')
        signerName: kubernetes.io/kube-apiserver-client
        expirationSeconds: 86400  # one day
        usages:
        - client auth

    EOF
    ```
    - for `.spec.request`, it should be a base64-encoded string of csr
      generated in the previous step, we use command to fetch it here:
      `cat user.csr | base64 | tr -d '\n'`
    - for `.spec.signerName`, it should be `kubernetes.io/kube-apiserver-client`
    - `.spec.expirationSeconds` is not required
    - `.spec.usages` should contain `client auth` only

1. once the CSR resource created in the cluster, we can check it with command
    ```
    kubectl get csr user-csr
    ```

    if there is no error, the output should be
    ```
    NAME         AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
    user-csr     2s    kubernetes.io/kube-apiserver-client   ${current-user}    24h                 Pending
    ```

1. to approve this CSR, use the following command:
    ```
    kubectl certificate approve user-csr
    ```

    and check the status of approved csr
    ```
    kubectl get csr user-csr
    ```
    the `CONDITION` would become `Approved,Issued`
    ```
    NAME         AGE   SIGNERNAME                            REQUESTOR          REQUESTEDDURATION   CONDITION
    user-csr     2s    kubernetes.io/kube-apiserver-client   ${current-user}    24h                 Approved,Issued
    ```

    or check it with the following command
    ```
    kubectl get csr user-csr -o yaml
    kubectl get csr user-csr -o json
    ```

1. so far, we have done the **authentication phase** for the new user, but the
   user don't have any permission in this cluster. Therefore we should complete
   the **authorization phase**

   to give the user permission in cluster, we should create a `Role` and
   `Rolebinding`, which is a important feature provided by K8s called **RBAC**

   for k3d-created cluster, there is a default **admin-permission**
   `ClusterRole` called **cluster-admin**, we can use it directly

1. create a ClusterRoleBinding for the new user
    ```
    kubectl create clusterrolebinding admin-user \
    --clusterrole=cluster-admin \
    --user=mashu
    ```

1. congrats! the new user become the cluster admin now!
   to be authenticated as the user, export the issued certificate first
    ```
    kubectl get csr user-csr -o jsonpath='{.status.certificate}'| base64 -d > user.crt
    ```

1. with this certificate, create a new user into kubeconfig
    ```
    kubectl config set-credentials ${USER_NAME} --client-key=user.key --client-certificate=user.crt --embed-certs=true
    ```
   replace the `USER_NAME` with your preferred name

   then, create a new context for that user
    ```
    kubectl config set-context ${CONTEXT_NAME} --cluster=${CLUSTER_NAME} --user=${USER_NAME}
    ```
   - replace `CONTEXT_NAME` with your preferred name
   - replace `CLUSTER_NAME` with your cluster name in kubeconfig
   - `USER_NAME` should be the same as you created in kubeconfig


## Note
1. the API version for CSR is `certificates.k8s.io/v1`

## References
[Official Doc](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests)

