# Generating the Data Encryption Config and Key
- Kubernetes stores a variety of data including cluster state, application configurations, and secrets. 
- Kubernetes supports the ability to encrypt cluster data at rest.

## The Encryption Key

Generate an encryption key and an encryption config suitable for encrypting Kubernetes Secrets.
```
ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)
```
```
echo $ENCRYPTION_KEY
```

## The Encryption Config File

Create the encryption-config.yaml encryption config file with the encryption key.

```
cat > encryption-config.yaml <<EOF
kind: EncryptionConfig
apiVersion: v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: "${ENCRYPTION_KEY}"
      - identity: {}
EOF
```

Copy the encryption-config.yaml encryption config file to each controller instance:
```
for instance in controller-0 controller-1 controller-2; do
  external_ip=$(aws ec2 describe-instances --filters \
    "Name=tag:Name,Values=${instance}" \
    "Name=instance-state-name,Values=running" \
    --output text --query 'Reservations[].Instances[].PublicIpAddress')
  
  scp -i kubernetes.id_rsa -o StrictHostKeyChecking=no encryption-config.yaml ubuntu@${external_ip}:~/
done
```