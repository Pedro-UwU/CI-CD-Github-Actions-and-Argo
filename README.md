# CI/CD with GitHub Actions & ArgoCD

This is a proof of concept of a CI/CD pipeline using GitHub Actions for continuous integration and ArgoCD for continuous deployment/delivery

## Prerequisites

- A Kubernetes cluster (I'll be using Minikube)
- Kubectl configured to use that cluster

## Steps

ArgoCD is a technology that covers the continuous deployment/distribution piece for our project with kubernetes. It automates the deployment of new k8s resources. What's great about ArgoCD is that it resides in our cluster, while listening to a repository where we have all our infrastructure as code. This follows GitOps, which is a set of practices to manage infrastructure and application configurations using Git. That means that to change the infrastructure of our cluster we only need to push the changes to a repository, and all will be automated!

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

Next, we'll need the admin account password. To get it, simply run and copy

```console
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

After that open a web browser and go to your ArgoCD server IP or URL In our case is 'https://localhost:8080/'.
Type 'admin' for the username and paste the password to log in.  
And that's it! ArgoCD is successfully installed in our kubernetes cluster!

## Create the infrastructure and the application

Next, we'll be using the [React Calculator](https://github.com/ahfarmer/calculator) from ahfarmer.
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

Replacing "\<your-registry-server\>" with the address of your registry server, "\<your-name\>" with your docker username, and "\<your-pword\>" with your docker password.

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
If all goes well, we should be able to see this:

![screenshot](https://github.com/PedroLopezITBA/CI-CD-Github-Actions-and-Argo/blob/main/images/app_sync.png?raw=true)

And we are done! To check the app, we need to expose the calculator-service. You can do this in many different ways, depending on if you are testing the app in your local machine or hosting it in a kubernetes cluster in the cloud. I'll use port-forwarding.

```console
kubectl -n calculator port-forward service/calculator-service 30000:3001
```

If we open 'http://localhost:30000' we should see a calculator!

![screenshot](https://github.com/PedroLopezITBA/CI-CD-Github-Actions-and-Argo/blob/main/images/calculator.png?raw=true)

## GitHub Actions

We made all the CD piece of our project and that's awesome! But what happens if we want to make a change to the calculator? We would need to create the image, push it into our private registry and change the tag of the image value in our deployment manifest. Here's where GitHub Actions comes into action, letting us automate this process. GitHub Actions is a technology that lets us automate our workflows. One advantage of Github Actions is that we can use other people's actions in our workflows, there is an entire marketplace of custom actions that we can use to avoid rewriting what other developers have already written.

### Create our workflows directory

GHA workflows are saved and stored in our repository. To enable them we need to create the `.github/workflows` directory. Inside this folder, we can create all the .yml files that we want. Each of these files represents a workflow, a workflow is a set of jobs which are a set of steps to be executed. Jobs run simultaneously, but can be instructed to run sequentially. These workflows might trigger depending on different events (e.g. every push, every merge, etc) and we can set their conditions to trigger. What's great about this is that we can create our own steps (like custom shell commands) or use other people's creations.  
To learn more about Github Actions check the [Official Documentation](https://docs.github.com/en/actions/quickstart)

### Design our workflow

First, we need to think about what we want GHA to do when we need to deploy a new version of our project.  
Based on what we've mentioned before, a good starting point is:

- Checkout our repo (Because GitHub Actions runs on cloud servers that are created when the job needs to be executed)
- Create the docker image with a new version tag
- Push the docker image to our private registry
- Update the 'calc-deploy.yml' file to have the new image tag
- Commit and push this tag change

This looks pretty nice, but there's another thing we have to decide, when will we trigger this workflow? We may think it's a good idea to trigger it every time we push something to the master branch, and maybe it is, but to avoid creating a ton of versions of our image, we can trigger the action every time we create a new tag, this way we can make a lot of updates/changes to our project and create a tag when we decide it's time to release a new version.

### Create the workflow

Let's get down to business to define the jobs. Here I'll show the yml file piece by piece and explain what each part means

```yml
# file name: create-image.yml
name: create-image
on:
  push:
    tags:
      - 'v*'
```

Here we declare the name of the workflow. After that, we indicate when we want the workflow to be triggered with the 'on' key value. By writing push and tags we indicate to GH that we want to execute the workflow every time a tag that starts with a 'v' is created.

```yml
jobs:
  create-and-push-image:
    runs-on: ubuntu-latest
    steps:
```

Here we indicate the jobs we want to run. The first one is called 'create and push image'. We indicate in what OS we want it to run and then we specify the steps of the job.

```yml
- name: get version
  id: get-version
  run: |
    echo ::set-output name=VERSION::$(echo $GITHUB_REF | cut -d / -f 3)
```

This is our first step. we named it 'get version' and gave it an id to get the output values later. The 'run' tag contains and echo command with the `::set-output` instruction, which sets an output value called VERSION that reads the value of the tag name. We will use it to write the tag of our docker image.

```yml
- name: login-docker
  run: echo ${{ secrets.JFROG_PASSWORD }} | docker login pedrolopez030200.jfrog.io -u ${{ secrets.JFROG_USERNAME }} --password
```

As the name suggests, this step if used for logging in to docker. It's important to realize that I'm using two GitHub secrets to access to my docker credentials.

```yml
- name: checkout
  uses: actions/checkout@v2
```

Here's our first action. As we can see, with this simple instruction we instruct GHA to checkout our repository.

```yml
- name: create image
  run: |
    cd app
    docker build -t test-app:${{ steps.get-version.outputs.VERSION }} .
    echo 'Created Image with name:tag = test-app:${{ steps.get-version.outputs.VERSION }}'
```

Here we go to the app directory to create the docker image. As we can see, I'm using the value of the VERSION output of `get version` to tag the image.

```yml
- name: get-images-id
  id: image-id
  run: |
    echo ::set-output name=IMAGE_ID::$(docker images -q test-app:${{ steps.get-version.outputs.VERSION }})
```

This step is used to get the image id of our newly created image, and save it in an output called IMAGE_ID. It is necessary for pushing the image to our private registry

```yml
- name: upload image to registry
  run: |
    docker tag ${{ steps.image-id.outputs.IMAGE_ID }} pedrolopez030200.jfrog.io/pedro-repo-docker/test-app:${{ steps.get-version.outputs.VERSION }}
    docker push pedrolopez030200.jfrog.io/pedro-repo-docker/test-app:${{ steps.get-version.outputs.VERSION }}  
```

And this is the moment we push the image to our registry. First, we tag it using the IMAGE_ID and VERSION, then we push it.

```yml
- name: install yq
  uses: mikefarah/yq@v4.15.1 

- name: update infra yaml file
  run: |
    cd infra
    yq e -i '.spec.template.spec.containers[0].image="pedrolopez030200.jfrog.io/pedro-repo-docker-local/test-app:${{ steps.get-version.outputs.VERSION }}"' calc-deployment.yml
    cat calc-deployment.yml
```

Once it's pushed, we need to change the version tag in our `calc-deployment.yml` file. To do so, we can use a tool called 'yq' (a YAML processor). First, we install it with an action from 'mikefatah' and then we go to the 'infra' directory and change it.

```yml
- name: push change
  uses: dmnemec/copy_file_to_another_repo_action@main
  env:
    API_TOKEN_GITHUB: ${{ secrets.API_TOKEN_GITHUB }}
  with:
    source_file: infra/calc-deployment.yml
    destination_repo: PedroLopezITBA/CI-CD-Github-Actions-and-Argo
    destination_branch: main
    destination_folder: infra
    user_email: pelopez@itba.edu.ar
    user_name: PedroLopezITBA
    commit_message: update image version ${{ steps.get_version.outputs.VERSION }} in yml file
```

And finally, the only thing left, is to push the change. Here we are using an action from dmnemec to copy and push a file to any repository. This is useful if we have our infrastructure as code in a separate repository. As we can see, we will need a GitHub secret with an active Github Token ([More Info](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)).  
And that is it! Our workflow is complete and after we push these changes it will run every time we create a new tag.

## Testing what we've done

Now we can test it by changing something in our calculator app (maybe the color of a button), pushing it, and seeing the workflow creating a pushing the image, for argoCD to detect that our cluster is out of sync. Let's do it.

I'll change a color value inside `app > src > component > Button.css`:

```css
...
.component-button.orange button {
  background-color: #0000FF; /*Change it to blue*/
  color: white;
}
...
```

After that, we'll create a new release in GitHub with the tag `v1.0.6` and this will trigger the workflow.
Once it is complete, it should push a change to our infra directory and argoCD will detect it.

![screenshot](https://github.com/PedroLopezITBA/CI-CD-Github-Actions-and-Argo/blob/main/images/argo_out_of_sync.png?raw=true)

Then, we proceed to Sync the application and we are done. After the pod finishes creating, we should be able to see the calculator with blue buttons. (You may need to restart the port-forwarding of the service if you are running this locally).

![screenshot](https://github.com/PedroLopezITBA/CI-CD-Github-Actions-and-Argo/blob/main/images/calc_blue.png?raw=true)

## Credits

- React Calculator: [Ahfarmer](https://github.com/ahfarmer/)
