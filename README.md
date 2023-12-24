# Coworking Space Service PROJECT

### Project Introduction
The Coworking Space Service is a set of APIs that enables users to request one-time tokens and administrators to authorize access to a coworking space. This service follows a microservice pattern and the APIs are split into distinct services that can be deployed and managed independently of one another.

For this project, you are a DevOps engineer who will be collaborating with a team that is building an API for business analysts. The API provides business analysts basic analytics data on user activity in the service. The application they provide you functions as expected locally and you are expected to help build a pipeline to deploy it in Kubernetes.

### Dependencies
#### Local Environment
1. Python 3.10+ and `pip`
2. Docker CLI - build and run Docker images locally
3. `kubectl` - run commands against a Kubernetes cluster
4. `helm` - apply Helm Charts to a Kubernetes cluster

#### Remote Resources
1. AWS CodeBuild - build Docker images remotely
2. AWS ECR - host Docker images
3. AWS EKS - run applications in k8s
4. AWS CloudWatch - monitor activity and logs in EKS
5. GitHub - pull and clone code
6. IAM Roles for EKS Cluster, Node Group, CodeBuild

### Instructions

#### A. Prepare resources
Create resources mentioned in <Remote Resources> using AWS Console/CLI/Terraform/Cloudformation

#### B. Create Postgre Database in EKS
Set up a Postgres database using a Helm Chart.

1. Set up Bitnami Repo & Install PostgreSQL Helm Chart
    ```bash
    helm repo add <REPO_NAME> https://charts.bitnami.com/bitnami
    helm install <SERVICE_NAME> <REPO_NAME>/postgresql
    ```

2. This should set up a Postgre deployment at `<SERVICE_NAME>-postgresql.default.svc.cluster.local` in your EKS cluster. You can verify it by running `kubectl svc`. By default, it will create a username `postgres`. The password can be retrieved with the following command:
    ```bash
    export POSTGRES_PASSWORD=$(kubectl get secret --namespace default <SERVICE_NAME>-postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
    echo $POSTGRES_PASSWORD
    ```

3. Test Database Connection (Optional)
    * Connecting Via Port Forwarding
    ```bash
    kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432 & PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
    ```

    * Connecting Via a Pod
    ```bash
    kubectl exec -it <POD_NAME> bash
    PGPASSWORD="<PASSWORD HERE>" psql postgres://postgres@<SERVICE_NAME>:5432/postgres -c <COMMAND_HERE>
    ```

4. Run Seed Files
Run the seed files in `db/` in order to create the tables and populate them with data.
```bash
kubectl port-forward --namespace default svc/<SERVICE_NAME>-postgresql 5432:5432
## (Open another terminal window and run below command for each .sql file in db/ ):
PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432 < <FILE_NAME.sql>
```

5. Running the Analytics Application Locally (Optional)
    In the `analytics/` directory:
    1. Install dependencies
    ```bash
    pip install -r requirements.txt
    ```
    2. Run the application (see below regarding environment variables)
    ```bash
    <ENV_VARS> python app.py
    ```
    with <ENV_VARS> includes:
    * `DB_USERNAME`
    * `DB_PASSWORD`
    * `DB_HOST` (defaults to `127.0.0.1`)
    * `DB_PORT` (defaults to `5432`)
    * `DB_NAME` (defaults to `postgres`)

    E.g:
    ```bash
    DB_USERNAME=username_here DB_PASSWORD=password_here python app.py
    ```

    3. Verifying The Application
    * Generate report for check-ins grouped by dates
    `curl <BASE_URL>/api/reports/daily_usage`

    * Generate report for check-ins grouped by users
    `curl <BASE_URL>/api/reports/user_visits`

6. Push code to Github to trigger CodeBuild process that automatically and pushed a built Docker image into ECR
    Verify Build history from CodeBuild and Image stored in ECR.

7. Deploy Kubernetes Coworking Space Service to EKS cluster
    # Update secrets and configmap before applying:
    kubectl apply -f ./eks_deploy/env_secrets.yaml
    kubectl apply -f ./eks_deploy/env_configmap.yaml
    # Service
    kubectl apply -f ./eks_deploy/apiapp_service.yaml
    # Deployment - Update the Dockerhub imagename and version in the deployment file according to image in ECR
    kubectl apply -f ./eks_deploy/apiapp_deployment.yaml
    
8. Expose the Deployment
    $DEPLOYMENT_NAME=coworkingspace-api-deployment
    kubectl expose deployment $DEPLOYMENT_NAME --type=LoadBalancer --name=public-api-for-coworkingspace-app

9. Test the exposed API:
    E.g:
    kubectl get svc
    a41a2ae3063ed407694e9d213ce02f9e-273838621.us-east-1.elb.amazonaws.com:5153/api/reports/user_visits
    a41a2ae3063ed407694e9d213ce02f9e-273838621.us-east-1.elb.amazonaws.com:5153/api/reports/daily_usage