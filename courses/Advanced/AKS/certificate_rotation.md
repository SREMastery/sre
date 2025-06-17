# Rotating AKS API Certificates

## Overview

Azure Kubernetes Service (AKS) uses certificates for secure communication between its components. These certificates need to be rotated periodically to maintain security. This guide provides detailed steps for rotating AKS API certificates.

## Prerequisites

1. Azure CLI installed and configured
2. `kubectl` installed
3. Access to the AKS cluster with appropriate permissions
4. Backup of your cluster configuration
5. Maintenance window scheduled (as this operation may cause brief service interruption)

## Step-by-Step Guide

### 1. Verify Current Certificate Status

```bash
# Get the current certificate information
kubectl get secret -n kube-system tls-ca -o yaml
```

### 2. Backup Current Configuration

```bash
# Create a backup directory
mkdir -p aks-backup-$(date +%Y%m%d)

# Backup current certificates
kubectl get secret -n kube-system tls-ca -o yaml > aks-backup-$(date +%Y%m%d)/tls-ca-backup.yaml
kubectl get secret -n kube-system tls-ca-key -o yaml > aks-backup-$(date +%Y%m%d)/tls-ca-key-backup.yaml
```

### 3. Rotate the Certificates

#### Method 1: Using Azure CLI

```bash
# Get your AKS cluster name and resource group
CLUSTER_NAME="your-cluster-name"
RESOURCE_GROUP="your-resource-group"

# Rotate the certificates
az aks rotate-certs --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP
```

#### Method 2: Manual Rotation

1. Generate new certificates:

```bash
# Generate new CA certificate
openssl genrsa -out ca.key 2048
openssl req -new -x509 -days 365 -key ca.key -out ca.crt -subj "/CN=Kubernetes CA"

# Generate new API server certificate
openssl genrsa -out apiserver.key 2048
openssl req -new -key apiserver.key -out apiserver.csr -subj "/CN=kube-apiserver"
openssl x509 -req -in apiserver.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out apiserver.crt -days 365
```

2. Update the secrets in Kubernetes:

```bash
# Create new secrets
kubectl create secret tls tls-ca --cert=ca.crt --key=ca.key -n kube-system
kubectl create secret tls tls-ca-key --cert=ca.key --key=ca.key -n kube-system
```

### 4. Verify the Rotation

```bash
# Check if the new certificates are in place
kubectl get secret -n kube-system tls-ca -o yaml

# Verify API server is responding
kubectl get nodes

# Check certificate expiration
kubectl get secret -n kube-system tls-ca -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates
```

### 5. Update Client Configurations

1. Update kubeconfig:

```bash
# Get new credentials
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --overwrite-existing
```

2. Update any automation tools or CI/CD pipelines that use the cluster credentials

## Best Practices

1. **Schedule Maintenance**

   - Plan the rotation during low-traffic periods
   - Notify all stakeholders in advance
   - Have a rollback plan ready

2. **Backup Strategy**

   - Always create backups before rotation
   - Store backups securely
   - Test backup restoration process

3. **Monitoring**

   - Monitor cluster health during and after rotation
   - Watch for any certificate-related errors
   - Check application connectivity

4. **Documentation**
   - Document the rotation process
   - Keep track of rotation dates
   - Maintain a history of certificate changes

## Troubleshooting

### Common Issues

1. **Certificate Mismatch**

   ```bash
   # If you see certificate mismatch errors
   kubectl get nodes
   # Error: x509: certificate signed by unknown authority
   ```

   Solution: Ensure all components are using the new certificates

2. **API Server Unreachable**

   ```bash
   # If API server becomes unreachable
   kubectl get pods
   # Error: Unable to connect to the server
   ```

   Solution: Check if the API server pod is running and verify network connectivity

3. **Client Authentication Issues**
   ```bash
   # If clients can't authenticate
   kubectl get pods
   # Error: x509: certificate has expired or is not yet valid
   ```
   Solution: Update client certificates and kubeconfig

### Rollback Procedure

If issues occur, you can rollback to the previous certificates:

```bash
# Restore from backup
kubectl apply -f aks-backup-$(date +%Y%m%d)/tls-ca-backup.yaml
kubectl apply -f aks-backup-$(date +%Y%m%d)/tls-ca-key-backup.yaml

# Restart the API server
kubectl delete pod -n kube-system -l component=kube-apiserver
```

## Security Considerations

1. **Certificate Lifetimes**

   - Set appropriate expiration dates
   - Plan for regular rotations
   - Monitor certificate expiration

2. **Access Control**

   - Limit access to certificate management
   - Use RBAC for certificate operations
   - Audit certificate changes

3. **Key Management**
   - Use strong key sizes (2048 bits minimum)
   - Secure private keys
   - Implement key rotation policies

## Automation

Consider automating the certificate rotation process:

```bash
#!/bin/bash
# Example automation script

# Variables
CLUSTER_NAME="your-cluster-name"
RESOURCE_GROUP="your-resource-group"
BACKUP_DIR="aks-backup-$(date +%Y%m%d)"

# Create backup
mkdir -p $BACKUP_DIR
kubectl get secret -n kube-system tls-ca -o yaml > $BACKUP_DIR/tls-ca-backup.yaml
kubectl get secret -n kube-system tls-ca-key -o yaml > $BACKUP_DIR/tls-ca-key-backup.yaml

# Rotate certificates
az aks rotate-certs --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP

# Verify rotation
kubectl get nodes
kubectl get secret -n kube-system tls-ca -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -dates

# Update credentials
az aks get-credentials --name $CLUSTER_NAME --resource-group $RESOURCE_GROUP --overwrite-existing
```

## Conclusion

Regular certificate rotation is crucial for maintaining the security of your AKS cluster. Follow these steps carefully, maintain proper documentation, and always have a rollback plan ready. Remember to test the process in a non-production environment first.

## Additional Resources

- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
- [Kubernetes Certificate Management](https://kubernetes.io/docs/tasks/tls/)
- [Azure CLI AKS Commands](https://docs.microsoft.com/en-us/cli/azure/aks)
