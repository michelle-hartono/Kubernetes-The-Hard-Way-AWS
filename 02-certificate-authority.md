# Certificate Authority
Provision a PKI Infrastructure using CloudFlare's PKI toolkit, cfssl, then use it to bootstrap a Certificate Authority, and generate TLS certificates for the following components: etcd, kube-apiserver, kube-controller-manager, kube-scheduler, kubelet, and kube-proxy.

## ✍️ Provisioning a CA 
CA is used to sign both client and server certificates, establishing a chain of trust.
1. Create Certificate Authority: `ca-config.json` 
    ```
    cat > ca-config.json <<EOF
    {
      "signing": {
        "default": {
          "expiry": "8760h"
        },
        "profiles": {
          "kubernetes": {
            "usages": [
              "signing",
              "key encipherment",
              "server auth",
              "client auth"
            ],
            "expiry": "8760h"
          }
        }
      }
    }
    EOF
    ```
    - This file defines:
        - How your Certificate Authority (CA) issues certificates
        - How long certificates are valid
        - What each certificate is allowed to do
2. Create Certificate Signing Request (CSR): `ca-csr.json`
    ```
    cat > ca-csr.json <<EOF
    {
      "CN": "kubernetes",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names":[{
        "C": "US",
        "ST": "Portland",
        "L": "Kubernetes",
        "O": "CA",
        "OU": "Oregon"
      }]
    }
    EOF
    ```
3. Initialize a Certificate Authority (CA) 
    ```
    cfssl gencert -initca ca-csr.json | cfssljson -bare ca
    ```
    - Create ca-key.pem (private key), ca.pem (public key), ca.csr (not always needed) 
    - ca-csr.json is for creating the CA 
    - ca-config.json is for using the CA to issue certs 

## 📝 Generating TLS Certificates (Using CA)
1. Admin Client Certificate
    - Create the configuration file (admin-csr.json)
        ```
        cat > admin-csr.json <<EOF
        {
          "CN": "admin",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [
            {
              "C": "US",
              "L": "Portland",
              "O": "system:masters",
              "OU": "Kubernetes The Hard Way",
              "ST": "Oregon"
            }
          ]
        }
        EOF
        ```
    - Generate the certificate
        ```
        cfssl gencert \
          -ca=ca.pem \
          -ca-key=ca-key.pem \
          -config=ca-config.json \
          -profile=kubernetes \
          admin-csr.json | cfssljson -bare admin
        ```
    - Results in `admin-key.pem` and `admin.pem`

2. Kubelet Client Certificates
    ```
    for i in 0 1 2; do
      instance="worker-${i}"
      instance_hostname="ip-10-0-1-2${i}"
      cat > ${instance}-csr.json <<EOF
    {
      "CN": "system:node:${instance_hostname}",
      "key": {
        "algo": "rsa",
        "size": 2048
      },
      "names": [
        {
          "C": "US",
          "L": "Portland",
          "O": "system:nodes",
          "OU": "Kubernetes The Hard Way",
          "ST": "Oregon"
        }
      ]
    }
    EOF

      external_ip=$(aws ec2 describe-instances --filters \
        "Name=tag:Name,Values=${instance}" \
        "Name=instance-state-name,Values=running" \
        --output text --query 'Reservations[].Instances[].PublicIpAddress')

      internal_ip=$(aws ec2 describe-instances --filters \
        "Name=tag:Name,Values=${instance}" \
        "Name=instance-state-name,Values=running" \
        --output text --query 'Reservations[].Instances[].PrivateIpAddress')

      cfssl gencert \
        -ca=ca.pem \
        -ca-key=ca-key.pem \
        -config=ca-config.json \
        -hostname=${instance_hostname},${external_ip},${internal_ip} \
        -profile=kubernetes \
        worker-${i}-csr.json | cfssljson -bare worker-${i}
    done
    ```
    - Kubelets must use a credential that identifies them as being in the system:nodes group to be authorized by the Node Authorizer, with a username of `system:node:<nodeName>`.
    - Results: 
        ```
        - worker-0-key.pem
        - worker-0.pem
        - worker-1-key.pem
        - worker-1.pem
        - worker-2-key.pem
        - worker-2.pem 
        ```
    
3. Controller Manager Client Certificate
    - Create the configuration file (kube-controller-manager-csr.json)
        ```
        cat > kube-controller-manager-csr.json <<EOF
        {
          "CN": "system:kube-controller-manager",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [
            {
              "C": "US",
              "L": "Portland",
              "O": "system:kube-controller-manager",
              "OU": "Kubernetes The Hard Way",
              "ST": "Oregon"
            }
          ]
        }
        EOF
        ```
    - Generate the certificate
        ```
        cfssl gencert \
          -ca=ca.pem \
          -ca-key=ca-key.pem \
          -config=ca-config.json \
          -profile=kubernetes \
          kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
        ```
    - Results in `kube-controller-manager-key.pem` and `kube-controller-manager.pem` 

4. Kube Proxy Client Certificate
    - Create the configuration file 
        ```
        cat > kube-proxy-csr.json <<EOF
        {
          "CN": "system:kube-proxy",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [
            {
              "C": "US",
              "L": "Portland",
              "O": "system:node-proxier",
              "OU": "Kubernetes The Hard Way",
              "ST": "Oregon"
            }
          ]
        }
        EOF
        ```
    - Generate the certificate
        ```
        cfssl gencert \
          -ca=ca.pem \
          -ca-key=ca-key.pem \
          -config=ca-config.json \
          -profile=kubernetes \
          kube-proxy-csr.json | cfssljson -bare kube-proxy
        ```
    - Results in `kube-proxy-key.pem` and `kube-proxy.pem`
    
5. Scheduler Client Certificate 
    - Create the config file
        ```
        cat > kube-scheduler-csr.json <<EOF
        {
          "CN": "system:kube-scheduler",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [
            {
              "C": "US",
              "L": "Portland",
              "O": "system:kube-scheduler",
              "OU": "Kubernetes The Hard Way",
              "ST": "Oregon"
            }
          ]
        }
        EOF
        ```
    - Generate the certificate
        ```
        cfssl gencert \
          -ca=ca.pem \
          -ca-key=ca-key.pem \
          -config=ca-config.json \
          -profile=kubernetes \
          kube-scheduler-csr.json | cfssljson -bare kube-scheduler
        ```
    - Results in` kube-scheduler-key.pem` and `kube-scheduler.pem` 

6. Kubernetes API Server Certificate
    - Define the DNS names used inside the cluster by pods and services when reaching the API server.
        ```
        KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local
        ```
    - Create the config file
        ```
        cat > kubernetes-csr.json <<EOF
        {
          "CN": "kubernetes",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [
            {
              "C": "US",
              "L": "Portland",
              "O": "Kubernetes",
              "OU": "Kubernetes The Hard Way",
              "ST": "Oregon"
            }
          ]
        }
        EOF
        ```
    - Generate the certificate
        ```
        cfssl gencert \
          -ca=ca.pem \
          -ca-key=ca-key.pem \
          -config=ca-config.json \
          -hostname=10.32.0.1,10.0.1.10,10.0.1.11,10.0.1.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
          -profile=kubernetes \
          kubernetes-csr.json | cfssljson -bare kubernetes
        ```
        > Specifying hostname (SANs): allows multiple identities (e.g., domain names, IP addresses) to be associated with a single certificate.
        > The name of the client (like kubectl, kubelets, or controllers) that connects to the API server must match what's in the cert’s Subject Alternative Names (SANs). 
    - Results in `kubernetes-key.pem` and `kubernetes.pem` 

7. Service Account Key Pair 
    - Create the config file 
        ```
        cat > service-account-csr.json <<EOF
        {
          "CN": "service-accounts",
          "key": {
            "algo": "rsa",
            "size": 2048
          },
          "names": [
            {
              "C": "US",
              "L": "Portland",
              "O": "Kubernetes",
              "OU": "Kubernetes The Hard Way",
              "ST": "Oregon"
            }
          ]
        }
        EOF
        ```
    - Generate the certificate
        ```
        cfssl gencert \
          -ca=ca.pem \
          -ca-key=ca-key.pem \
          -config=ca-config.json \
          -profile=kubernetes \
          service-account-csr.json | cfssljson -bare service-account
        ```
    - Results in `service-account-key.pem` and `service-account.pem` 

8. Distribute the Client and Server Certificates
    - Copy the appropriate certificates and private keys to each worker instance:
        ```
        for instance in worker-0 worker-1 worker-2; do
          external_ip=$(aws ec2 describe-instances --filters \
            "Name=tag:Name,Values=${instance}" \
            "Name=instance-state-name,Values=running" \
            --output text --query 'Reservations[].Instances[].PublicIpAddress')

          scp -i kubernetes.id_rsa -o StrictHostKeyChecking=no ca.pem ${instance}-key.pem ${instance}.pem ubuntu@${external_ip}:~/
        done
        ```
    - Copy the appropriate certificates and private keys to each controller instance
        ```
        for instance in controller-0 controller-1 controller-2; do
          external_ip=$(aws ec2 describe-instances --filters \
            "Name=tag:Name,Values=${instance}" \
            "Name=instance-state-name,Values=running" \
            --output text --query 'Reservations[].Instances[].PublicIpAddress')

          scp -i kubernetes.id_rsa -o StrictHostKeyChecking=no \
            ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
            service-account-key.pem service-account.pem ubuntu@${external_ip}:~/
        done
        ```

The kube-proxy, kube-controller-manager, kube-scheduler, and kubelet client certificates will be used to generate client authentication configuration files in the next lab.