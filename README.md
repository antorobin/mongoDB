# MongoDB Kubernetes Deployment with HAProxy

This repository contains Kubernetes manifests to deploy MongoDB with HAProxy for high availability. The setup uses Azure Disk for persistent storage.

## Prerequisites
- Kubernetes cluster running on Azure
- `kubectl` installed and configured
- Helm installed (optional, for managing deployments)

## Deployment Steps

### Step 1: Apply Kubernetes Manifests

1. Create a namespace:
   ```sh
   kubectl create namespace mongo-namespace
   ```

2. Apply the secret for MongoDB credentials:
   ```sh
   kubectl -n mongo-namespace apply -f mongo-credentials-secret.yaml
   ```

3. Create the storage class for Azure Disk:
   ```sh
   kubectl -n mongo-namespace apply -f mongo-storageclass-azure-disk.yaml
   ```

4. Deploy MongoDB StatefulSet and Service:
   ```sh
   kubectl -n mongo-namespace apply -f mongo-statefulset.yaml
   kubectl -n mongo-namespace apply -f mongo-service.yaml
   ```

5. Initialize the MongoDB replica set:
   ```sh
   kubectl exec -it mongodb-0 -n mongo-namespace -- mongosh
   ```
   Inside the MongoDB shell, run:
   ```js
   rs.initiate(
     {
       _id: "rs0",
       members: [
         { _id: 0, host: "mongodb-0.mongodb-service.mongo-namespace.svc.cluster.local:27017" },
         { _id: 1, host: "mongodb-1.mongodb-service.mongo-namespace.svc.cluster.local:27017" },
         { _id: 2, host: "mongodb-2.mongodb-service.mongo-namespace.svc.cluster.local:27017" }
       ]
     }
   )
   ```

6. Create an admin user:
   ```js
   use admin
   db.createUser({
     user: "admin",
     pwd: "demo",
     roles: [
       { role: "userAdminAnyDatabase", db: "admin" },
       { role: "dbAdminAnyDatabase", db: "admin" },
       { role: "readWriteAnyDatabase", db: "admin" },
       { role: "clusterAdmin", db: "admin" }
     ]
   })
   ```

### Step 2: Deploy HAProxy

1. Apply HAProxy ConfigMap and Deployment:
   ```sh
   kubectl -n mongo-namespace apply -f haproxy-config.yaml
   kubectl -n mongo-namespace apply -f haproxy-deployment.yaml
   ```

2. Expose HAProxy as a service:
   ```sh
   kubectl -n mongo-namespace apply -f haproxy-internal-service.yaml
   kubectl -n mongo-namespace apply -f haproxy-external-service.yaml
   ```

### Step 3: Connect to MongoDB via HAProxy

Get the HAProxy public IP address:
   ```sh
   kubectl get svc -n mongo-namespace
   ```

Connect using MongoDB shell or Compass:
   ```sh
   mongosh "mongodb://admin:demo@<haproxy-public-ip>:9291/?authSource=admin&directConnection=true"
   ```

## Reference Article
For a detailed explanation of the setup, refer to the Medium article:
[Deploying MongoDB with HAProxy in Kubernetes](https://medium.com/@antorobin/high-availability-mongodb-deployment-on-aks-with-haproxy-step-by-step-guide-1938767a813d)

## Conclusion
This setup provides a highly available MongoDB cluster with read/write load balancing via HAProxy.