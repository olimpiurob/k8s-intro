Kubernetes Movie Application Deployment Test

## Overview

In this test, you will deploy a multi-tier movie application in Kubernetes. This application consists of three main components:
1. A frontend web interface
2. A backend API
3. A MySQL database

You will need to create and configure several Kubernetes resources to make this application function correctly, including deployments, services, configuration, and proper routing.

## Requirements

### Docker Images
You must use the following Docker images:
- Frontend: `lbogdan/movies:v2`
- Backend: `lbogdan/movies-api:v6`
- Database: `mysql:5.7.40`

### Components to Create

1. **Database Layer (MySQL)**:
   - StatefulSet for MySQL
   - Persistent storage for database data
   - Service to expose MySQL internally

2. **Backend API**:
   - Deployment for the API service
   - Service to expose the API
   - Configuration for database connectivity
   - Job to initialize the database

3. **Frontend Web UI**:
   - Deployment for the web interface
   - Service to expose the frontend

4. **Configuration**:
   - ConfigMap containing database connection info
   - Secret containing database credentials
   - Ingress rules for routing external traffic

## Specific Tasks

### 1. Create Database Secret

Create a Kubernetes Secret named `movies-secret` that includes the following values:
- `DATABASE`: "movies"
- `DATABASE_USER`: "$username"
- `DATABASE_PASSWORD`: "yoursecretpassword"

### 2. Create Configuration

Create a ConfigMap named `config-json` that contains a JSON configuration file with the following structure:

```json
{
  "database": {
    "host": "movies-mysql-svc",
    "name": "movies"
  }
}
```

This config should be mounted at `/app/config` in both the backend deployment and database initialization job.

### 3. Deploy MySQL Database

Create a StatefulSet for MySQL with:
- Appropriate resource requests and limits
- Persistent storage for database files
- Required environment variables sourced from the secret
- A service named `movies-mysql-svc` to expose it on port 3306

### 4. Create Database Initialization Job

Create a Job that:
- Uses the backend image (`lbogdan/movies-api:v6`)
- Runs the `db-init` command
- Has access to the database credentials from the secret
- Has the config volume mounted

### 5. Deploy Backend API

Create a Deployment for the backend API that:
- Uses the `lbogdan/movies-api:v6` image
- Exposes port 8000
- Includes the database credentials as environment variables
- Mounts the configuration
- Has a corresponding Service named `movies-backend-svc`

### 6. Deploy Frontend

Create a Deployment for the frontend that:
- Uses the `lbogdan/movies:v2` image
- Exposes port 80
- Has a corresponding Service named `movies-frontend-svc`

### 7. Configure Ingress

Create an Ingress resource that:
- Routes traffic to the frontend service at the root path (`/`)
- Routes traffic to the backend service at the `/api` path

## Success Criteria

Your deployment is successful when:
1. The database has been initialized successfully
2. The frontend website loads and displays movies
3. The frontend can communicate with the backend API
4. Data is persisted in the MySQL database

## Hints

- Make sure the backend can connect to the database using the provided config
- The backend API needs to be accessible at the `/api` path for the frontend to work
- All three tiers (frontend, backend, database) need to communicate with each other
- Check pod logs if you encounter any issues
- You can create a debug pod to troubleshoot database connectivity

NOTE: You will be facing an issue with the mysql pod, since the rook-ceph provisioner will provision a volume containing a LOST+FOUND directory, which MySQL does not like. You will need to delete this directory from the volume before MySQL can start successfully.
- You can use `kubectl exec` to run commands inside the pods for debugging or create a debug pod mounting the same volume as the MySQL pod.
- Use `kubectl describe` and `kubectl logs` to troubleshoot issues with your deployments and services.

Good luck!
