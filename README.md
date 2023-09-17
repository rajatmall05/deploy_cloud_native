# deploying_cloud_native_monitoring_app_on_k8s
Prerequisites:
-> AWS Account
-> IAM Access and AWS configured with CLI
-> Python 3
-> Docker and Kubernetes installed
-> Any Code Editor (VScode)


Stage 1: Deploying the Flask app locally

        Step 1: Install dependencies {Flask, MarkupSafe, Werkzeug, itsdangerous, psutil, plotly, tenacity, boto3, kubernetes}
        Command:pip3 install -r requirements.txt
        
        Step 2: Run the app
        Command:python3 app.py
        It will start the Flask server on localhost:5000

Stage 2: Dockerizing the Flask application

        Step 1: Create a Dockerfile in the root directory with the following commands
        Command:FROM python:3.11-buster
                WORKDIR /app
                COPY requirements.txt .
                RUN pip3 install --no-cache-dir -r requirements.txt
                COPY . .
                ENV FLASK_RUN_HOST=0.0.0.0
                EXPOSE 5000
                CMD ["flask", "run"]

        Step 2: Build the docker image
        Command:docker build -t <img-name> .

        Step 3: Run the docker container
        Command:docker run -p 5000:5000 <img-name>
        It will start the Flask server in Docker container on localhost 5000

Stage 3: Push Docker image to Elastic Container Registry(ECR)
        
        Step 1: Create an ECR Repo
        Command:import boto3
                ecr_client = boto3.client('ecr')
                repository_name = "monitoring_image"
                response = ecr_client.create_repository(repositoryName=repository_name)
                repository_uri = response ['repository']['repositoryUri']
                print(repository_uri)

        Step 2: Push Docker image to ECR
        Command:docker push <ecr_repo_uri>:<tag>

Stage 4: Creating an EKS Cluster and deploying the app using Python 
        
        Step 1: Create an EKS Cluster and add node group
        Step 2: Create a node group in EKS Cluster
        Step 3: Create deployment and service        
        Command:from kubernetes import client, config
                # Load Kubernetes configuration
                config.load_kube_config()
                # Create a Kubernetes API client
                api_client = client.ApiClient()
                # Define the deployment
                deployment = client.V1Deployment(
                    metadata=client.V1ObjectMeta(name="flask-app"),
                    spec=client.V1DeploymentSpec(
                        replicas=1,
                        selector=client.V1LabelSelector(
                            match_labels={"app": "flask-app"}
                        ),
                        template=client.V1PodTemplateSpec(
                            metadata=client.V1ObjectMeta(
                                labels={"app": "flask-app"}
                            ),
                            spec=client.V1PodSpec(
                                containers=[
                                    client.V1Container(
                                        name="flask-container",
                                        image="568373317874.dkr.ecr.ap-south-1.amazonaws.com/my_monitoring_app_image:latest",
                                        ports=[client.V1ContainerPort(container_port=5000)]
                                    )
                                ]
                            )
                        )
                    )
                )
                
                # Create the deployment
                api_instance = client.AppsV1Api(api_client)
                api_instance.create_namespaced_deployment(
                    namespace="default",
                    body=deployment
                )
                # Define the service
                service = client.V1Service(
                    metadata=client.V1ObjectMeta(name="flask-service"),
                    spec=client.V1ServiceSpec(
                        selector={"app": "flask-app"},
                        ports=[client.V1ServicePort(port=5000)]
                    )
                )
                # Create the service
                api_instance = client.CoreV1Api(api_client)
                api_instance.create_namespaced_service(
                    namespace="default",
                    body=service
                )

        Once you hit "python3 elastic_kubernetes_service.py", deployment and service will be created.
        Finally check by running following comands:
        
        Command:kubectl get deployment -n default (check deployments)
                kubectl get service -n default (check service)
                kubectl get pods -n default (to check the pods)
        Once pod is up and running, run port-forward to expose the service:
        Command:kubectl port-forward service/<service_name> 5000:5000
        
