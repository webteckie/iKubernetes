# iKubernetes
Describes some kubernetes learning resources. For this we will use the walk through at the following link as the basis for this learning:

- [Kubernetes Docs](https://kubernetes.io/)
- [Kubernetes in 5 mins](https://www.youtube.com/watch?v=PH-2FfFD2PU)
- [Kubernetes Tutorial For Beginners](https://www.youtube.com/watch?v=F-p_7XaEC84)
- [Kubernetes Hands-on Tutorial](https://docs.microsoft.com/en-us/azure/aks/tutorial-kubernetes-prepare-app)

You can certainly follow all the instructions at that link and hopefully you will have a working system at the end. But this README summarizes the necessary steps to ensure a working application. 

NOTE: Ensure that you have downloaded and installed Docker in your local environment.

NOTE: As of this writing, although everything worked with the use of an Azure ACR registry, the final app deployment did not
seem to work due to permissions with using someone else's Kubernetes cluster and not having all the necessary permissions setup.
As a result, the exercise finished successfully by using the Docker Hub image repository instead.


## Azure CLI
The Azure CLI is how most of the commands to Azure are performed.

- Installed the Azure CLI using PowerShell (Run as Admin)

    Invoke-WebRequest -Uri https://aka.ms/installazurecliwindows -OutFile .\AzureCLI.msi; Start-Process msiexec.exe -Wait -ArgumentList '/I AzureCLI.msi /quiet'

- Onced installed closed and reopen Powershell


## Clone Voting Application
This exercise uses the voting application at the established tutorial link although you can use any application you want as long as you have the neccessary docker-compose and kubernetes deploy YAML files. Once you clone the application it may be useful to open a Powershell session to the root directory of that application so when it is time to issue commands that use the .yaml file you are already there. Or, even better, open the application in VS Code and then within VS Code open a Powershell terminal window to run all commands.


## Build the application Docker image

- Build and run the image locally:

    docker-compose up -d

- Ensure the image was created:

    docker images


## The Image Registry
Since the most basic deployable unit of work to a kubernetes cluster is a docker image, we need to have a docker image
registry to host these. The most popular and default option is [docker hub](hub.docker.com) But Azure also has a 
Container Registry (a.k.a ACR). Here we will describe both registries.


## Setting up a Kubernetes cluster in Azure
This part is TBD. An existing cluster was used for this exercise. To use someone else's cluster do:

    az account set --subscription "{the-subscription-id-where-the-cluster-resides}"

    az aks get-credentials --name {existing-cluster-name} --resource-group {its-resource-group}

This should update the kubernetes config to allow usage of that cluster. Should see something like:
    Merged "{existing-cluster-name}" as current context in C:\Users\{username}\.kube\config

If multiple people are working in the cluster then each can have a namespace to work under by doing:

    kubectl config set-context --current --namespace={namespace-name}

To create a new cluster something like the following should work:
    
    az aks create --resource-group {existing-resource-group-name} --name {cluster-name} --node-count 2 --generate-ssh-keys


## Working the the Azure ACR container registry 

- Ensure you have an account with sufficient priviledges to create resources in Azure

- Create the ACR

    az acr create --resource-group cb-epts --name {acr-name} --subscription {subscription} --sku Basic

    NOTE: this step can also be performed via the azure portal GUI.

- Login to ACR
    az acr login --name {acr-name} --username {acr-username} --password {acr-password}

    NOTE: the {acr-username} and {acr-password} you can get by navigating to the ACR in Azure and then viewing
    the Access Keys panel. Ensure the "Admin user" in that panel is enabled for this to work!

- Tag the app image with a version

    docker tag azure-vote-front {acr-login-server}/azure-vote-front:v1

    NOTE: the {acr-login-server} you can get by navigating to the ACR in Azure and then viewing
    the Overview panel.

- Ensure the image with new tag shows by running:

    docker images

- Push the image to the ACR

    docker push {acr-name}.azurecr.io/azure-vote-front:v1

- List images in the ACR

    az acr repository list --name {acr-name} --username {acr-username} --password {acr-password} --output table

## Generate Secret for ACR access
The Azure ACR images are private and thus requires a secret token for the Kubernetes service to access the images. 

- To generate that token do the following:

    kubectl create secret docker-registry {secret-name} --docker-server={acr-login-server} --docker-username={acr-username} --docker-password={acr-password}

- Verify secrets stored in the Kubernetes service:

    kubectl get secrets

    NOTE: the secret with name {secret-name} should show.

- You can also get details about the secret:

    kubectl describe secret {secret-name}


## Working the the Docker Hub container registry 
By default kubernetes deployments assume a Docker Hub container registry, if a fully qualified one 
(e.g., myacr.azureacr.io) is not specified. Here we describe how to setup your images in Docker Hub
to be used as the registry for the Kubernetes cluster.

- Ensure you have an account in Docker Hub

- Login to your Docker Hub account both via the browser and also via the local docker desktop

    NOTE: to login your local docker desktop go to the settings and then do signin there.

- Create the image repository (e.g., azure-vote-front) navigating to the Repositories menu and using "Create Repository" button.

    NOTE: for this exercise ensure the created repository is public!

- Tag the app image with a version

    docker tag azure-vote-front {your-docker-hub-username}/azure-vote-front:v1

- Ensure the image with new tag shows by running:

    docker images

- Push the image to Docker Hub

    docker push {your-docker-hub-username}/azure-vote-front:v1

- Ensure the tagged image shows in Docker Hub by navigating to the repository in Docker Hub


## Deploying the application
Before you deploy the application you need to modify the azure-vote-all-in-one-redis.yaml in the cloned app to ensure the container image repository for the azure-vote-front app is correctly set. 

- For Docker Hub ensure that is set to:

      containers:
      - name: azure-vote-front
        image: {your-docker-hub-username}/azure-vote-front:v1

- For the Azure ACR ensure that is set to:

      containers:
      - name: azure-vote-front
        image: {acr-login-server}/azure-vote-front:v1
        ...
      imagePullSecrets:
        - name: {secret-name}

    NOTE: ensure that imagePullSecrets is at the same indentation level as containers and that it is specified after all the container properties!!!

Once that is correctly set and saved you can actually perform the deploy of the app to the Kubernetes cluster:

    kubectl apply -f azure-vote-all-in-one-redis.yaml

- Ensure the application is deployed:

    kubectl get pods

    NOTE: should see something like:

    NAME                                READY   STATUS    RESTARTS   AGE
    azure-vote-back-6d4b4776b6-nf7r6    1/1     Running   0          16h
    azure-vote-front-647868dd6c-2fwmn   1/1     Running   0          5m33s

- You can get a description of the azure-vote-front pod by doing:

    kubectl describe pod azure-vote-front-{XXXXX-YYYYY}


## Browse to the app
If and after the application was successfully deployed then you should be able to browse to it.

- Get the external IP address:

    kubectl get service azure-vote-front --watch

- Browse to the external IP address in a browser and the voting app should display


Have fun voting!
