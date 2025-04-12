# Configuring kubectl for Remote Access
 - Generate a kubeconfig file for the kubectl command line utility based on the admin user credentials.
- Run the commands in this lab from the same directory used to generate the admin client certificates.

## The Admin Kubernetes Configuration File
- Each kubeconfig requires a Kubernetes API Server to connect to. 
- To support high availability the IP address assigned to the external load balancer fronting the Kubernetes API Servers will be used.

1. Generate a kubeconfig file suitable for authenticating as the admin user:
    ```
    KUBERNETES_PUBLIC_ADDRESS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns ${LOAD_BALANCER_ARN} \
    --output text --query 'LoadBalancers[].DNSName')

    kubectl config set-cluster kubernetes-the-hard-way \
      --certificate-authority=ca.pem \
      --embed-certs=true \
      --server=https://${KUBERNETES_PUBLIC_ADDRESS}:443

    kubectl config set-credentials admin \
      --client-certificate=admin.pem \
      --client-key=admin-key.pem

    kubectl config set-context kubernetes-the-hard-way \
      --cluster=kubernetes-the-hard-way \
      --user=admin

    kubectl config use-context kubernetes-the-hard-way
    ```
    Output
    ```
    Cluster "kubernetes-the-hard-way" set.
    User "admin" set.
    Context "kubernetes-the-hard-way" created.
    Switched to context "kubernetes-the-hard-way".
    ```
2. Verification
    * Check the version of the remote Kubernetes cluster 
        ```
        kubectl version
        ```
        Output
        ```
        Client Version: v1.30.5
        Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
        Server Version: v1.28.3
        ```
    * List the nodes in the remote Kubernetes cluster
        ```
        kubectl get nodes
        ```
        Output
        ```
        NAME           STATUS   ROLES    AGE   VERSION
        ip-10-0-1-20   Ready    <none>   15m   v1.28.3
        ip-10-0-1-21   Ready    <none>   15m   v1.28.3
        ip-10-0-1-22   Ready    <none>   15m   v1.28.3
        ```
