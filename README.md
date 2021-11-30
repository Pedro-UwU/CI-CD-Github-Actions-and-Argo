# CI/CD with GitHub Actions & ArgoCD

This is a proof of concept of a CI/CD pipeline using GitHub Actions for continuous integration and ArgoCD for continuous deployment/delivery

## Prerequisites

- A Kubernetes cluster (I'll be using Minikube)
- Kubectl configured to use that cluster

## Steps

### Install ArgoCD

[Reference](https://argo-cd.readthedocs.io/en/stable/getting_started/)

Run:

```console
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

I'm testing this in my local machine, therefore, I'll use port-forwarding to connect to the API server. 
See 'Reference' to know how to expose the API server with another method.

In **another** terminal, run:

```console
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Next, I'll need the admin account password. To get it, simply run and copy

```console
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

After that open a web browser and go to your ArgoCD server IP or URL In my case is 'https://localhost:8080/'.
Type 'admin' for the username and paste the password to log in.  
And that's it! ArgoCD is successfully installed in our kubernetes cluster!

## Create an infrastructure and the application

Next, we'll be using the [React Calcultor](https://github.com/ahfarmer/calculator) from ahfarmer.
The project is located in the 'app' directory of this repository.

### Write IaC

We'll be creating a simple deployment and service to host a simple calculator application.
First, let's take a look at the deployment file:

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: calculator
  name: calculator-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: calculator
  template:
    metadata:
      labels:
        app: calculator
    spec:
      containers:
      - image: pedrolopez030200.jfrog.io/pedro-repo-docker-local/test-app:v1.0.3
        name: calculator-container
        ports:
        - containerPort: 3000
      imagePullSecrets:
      - name: regcred
```

As we can see, we are creating a deployment resource within the 'calculator' application. It will only create one replica of each pod.
The containers image is pulled from a private repository, and it will pull the credentials from a secret called regcred (More on that later).
Feel free to change the image to your own image in a private repository or a public one from docker hub.

Then, let us see the service file

```yml
apiVersion: v1
kind: Service
metadata:
  name: calculator-service
spec:
  selector:
    app: calculator
  ports:
    - protocol: TCP
      port: 3001
      targetPort: 3000
```

Here we are creating a Cluster-IP service that targets the port 3000 of our pods with the `app=calculator` label.

### Credentials in a secret

We need the credentials of our private registry to get the docker image. We can store them in a secret object, but we are working with GitOps. That means that we will expose our secrets in our repository, and that's a security flaw. This is a very discussed problem every time someone brings up GitOps and some solutions have been proposed.  
Luckily we can use a very simple solution. We will create and inject our secret into the 'calculator' namespace via kubectl. ArgoCD won't delete and our k8s resources will be able to use it. This is not an elegant nor scalable solution, but it will work for our purposes.

First, we need to create a namespace for our app and our secret:

```console
kubectl create namespace calculator
```

To do this we have to type the following command:

```console
kubectl -n calculator create secret docker-registry regcred --docker-server=<your-registry-server> --docker-username=<your-name> --docker-password=<your-pword>
```

Replacing "\<your-registry-server\>" with the address of your registry server, "\<your-name>\>" with your docker username, and "\<your-pword\>" with your docker password.

To check if the secret has been created, we can run the following command:

```console
kubectl -n calculator get secret regcred -o yaml
```

This will output our secret and we can check if the data is correct.

### Create the app

Go to the ArgoCD UI and click the 'New App' button.

- Write the name you desire for this app, I'm choosing 'calculator-app' (it can only be lowercase letters and hyphens)
- Select the 'dafult' project
- Check 'AUTO-CREATE NAMESPACE'
- Copy and paste the URL of the repository with our IaC. I'll use this repository URL.
- Write the path of the yml files in the 'Path' text box. In this case, is './infra'
- For 'Cluster URL' select the local cluster ('https://kubernetes.default.svc')
- Choose a namespace. I'll use 'calculator'.  It must be the same namespace as our secret

After that, click on 'CREATE' and wait for the app to be created. We will see something like this:

![screenshot](https://github.com/PedroLopezITBA/CI-CD-Github-Actions-and-Argo/blob/main/images/app_created.png?raw=true)

Which is great! It means that ArgoCD found our repository and all went smoothly.  
Now the moment of truth, click 'SYNC' and then synchronize the app.
If all went well, we should be able to see this:

![screenshot](https://github.com/PedroLopezITBA/CI-CD-Github-Actions-and-Argo/blob/main/images/app_sync.png?raw=true)

And we are done! To check the app, we need to expose the calculator-service. You can do this in many different ways, depending on if you are testing the app in your local machine or hosting it in a kubernetes cluster in the cloud. I'll use port-forwarding.

```console
kubectl -n calculator port-forward service/calculator-service 30000:3001
```

If we open 'http://localhost:30000' we should see a calculator!

