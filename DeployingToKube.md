# Deploying to Kubernetes using Docker and Helm

Everything we have built runs locally, but using `kitura init`s Helm charts along with Docker for Desktops Kubernetes support will allow us to deploy to a local cluster, mimicking deployment to a cloud environment such as IBM Cloud. In IBM Cloud, we would gain support for scaling our app depending on demand, containerisation to keep our app running smoothly if anything goes wrong, load balancing, rolling updates and more!

## Pre-Requisites

Ensure you have installed [Docker for Desktop](https://www.docker.com/products/docker-desktop) on your Mac and enabled Kubernetes within the app. To do so, select the docker icon in the Menu Bar and click Preferences > Kubernetes Tab > Enable Kubernetes. It will take a few moments to install and start up, but when the indicator light in the bottom right corner is green, you're ready to go!

You are also going to need to install Helm using brew:

```
brew install kubernetes-helm
```

![docker-menubar](./resources/docker-menubar.png)

![docker-app](./resources/docker-app.png)

## Deploying a PostgreSQL database instance

The entire application will consist of two pods, one for the database and one for the Kitura server running your Swift code. We will start by deploying a PostgreSQL instance to your local Kubernetes cluster using a Helm chart.

### Adding repos to Helm

Helm has a repo named `stable` which has a load of commonly used charts (blueprints to things you might want to run in a cluster, like databases!), but we need to tell Helm where to look for these blueprints.

```bash
helm init #Initialises Helm on our newly created Kubernetes cluster
helm repo add stable https://kubernetes-charts.storage.googleapis.com
```

### Creating the database 

We can now create a PostgreSQL instance called `postgresql-database`. 

**Note**: Kubernetes is very specific on names, and calling your database anything else will result in later parts of the tutorial requiring tweaking.

```bash
helm install --name postgresql-database stable/postgresql
```

There should be output immediately, and we need to run some of the commands listed. Specifically, we need the user password. Copy and paste the command beginning with `PGPASSWORD=` and run it. Connect to your database to check the password works by running the command beginning with `kubectl run` and ensure a prompt appears.

When the prompt appears, exit using `\q`. Now would be a good time to note the password, as we will need to update our Swift code to use it. 

```bash
echo $PGPASSWORD #Note down the output
```

We must now create a database within the postgresql instance, similar to what we did in the previous tutorial. PostgreSQL is running inside a Docker container, which lives inside your Kubernetes cluster, so we need to get into the Docker container running PostgreSQL. We need the Pod name for the next command, which is randomly assinged when the Pod is generated.

```bash
kubectl get pods #Copy the value under NAME prefixed by postgresql-database
kubectl exec -it <COPIED-POD-NAME> bash
```

We are now inside the Docker container, denoted by your prompt changing to `root@COPIED-POD-NAME`. We can now use the same command from the last tutorial to create our database.

```bash
createdb tododb
exit #Exits the Docker container
```

We now have a database inside your Docker container, running within a Kubernetes cluster. Well done!

## Edit the Swift code

### 1. Edit the Swift code for connection to Kubernetes

Your Swift code is going to need some minor changes to reflect using a database inside the local cluster. All the changes are happening in the declaration within your `Persistence` class, inside the `let pool =` assignment. Here is where the password for the database you noted in step 2 will be used.

Change `host:` from `localhost` to `postgresql-database`. Keep port as `5432`

Leave `.databaseName("tododb") ` as is, but add some new options to the array. We need to add `.password("VALUE YOU NOTED DOWN EARLIER")` and `.userName("postgres")`. Make sure each element is separated with a comma, and all the values you entered are type `String`. 

These changes allow the Kitura sever to access the database and write/read from it.

## Docker

### Build the code into local Docker images

We are now ready to compile your code and create a Docker image, ready to deploy to your local cluster. `kitura init` provides us with the tools, but we need to modify the `Dockerfiles` before we build anything. Open up Dockerfile and change the line

```
# RUN apt-get update && apt-get dist-upgrade -y
```
and modify it to say the following. Make sure to remove the comment in front of `RUN`
```dockerfile
RUN sudo apt-get update && apt-get install -y libpq-dev
```

Without this, our Swift code would not compile as the Docker container would be lacking the necessary system packages. 

Repeat the same for `Dockerfile-tools`. When you have completed this, we can build our images. These commands require pulling from a remote repo, and may take a few moments to complete.

```bash
docker build -t server-run .
docker build -t server-build -f Dockerfile-tools .
docker tag server-run:latest server-run:1.0.0
```

This builds us our images and tags the executable container, a requirement for using our Helm chart. We can now compile the code inside the image to make it ready for the cluster.

```bash
docker run -v $PWD:/swift-project -w /swift-project server-build /swift-utils/tools-utils.sh build release
```

Now our `server-run` docker image has our Swift code as an executable which can be deployed to Kuberntetes using the Helm chart that `kitura init` provided us with!

## Deploy with Helm

### Using Helm charts to deploy to Kubernetes

First we must edit the chart that will be used to deploy to Kubernetes to point to our local, tagged `server-run` image. Using Xcode's left sidebar, navigate to chart > ToDoServer > 

This chart acts like a blueprint for Helm to use when deploying our application to Kubernetes. We need to modify the `repository`, `tag` and `pullPolicy` lines (towards the top of the file).

```yaml
...
image:
    repository: server-run
    tag: 1.0.0
    pullPolicy: IfNotPresent
...
```

We are telling Helm to use a local Docker image called `server-run` tagged at `1.0.0`, and to only pull from a remote if we can't find the image locally.

We are now ready to deploy our Helm chart into Kubernetes.

```bash
helm install --name server chart/ToDoServer
kubectl get all #Ensure the pod todoserver-deployment STATUS is Running
```

Now everything is up and running! To access the database, we will use the OpenAPI UI route.

### Accessing the Application from a browser

We can't navigate to `localhost:8080` as usual because our cluster isn't part of the localhost network. Port forwarding is built into the `kubectl` tools, and allows us to access our application from a browser as usual.

```bash
kubectl get pods #Copy the todoserver NAME
kubectl port-forward <todoserver-deployment-XXXXXX-XXXXX> 8080:8080
```

We can now open a browser, and go to [localhost:8080/openapi/ui](localhost:8080/openapi/ui) where the OpenAPI dashboard should display. Using the drop down menus, select `POST /` and click the Example Value on the right hand side to autofill the input field. Now select `Try it out!` and a `201` response should be recieved (you may need to scroll down to see it).

Now try `GET /` and the response should `200` and the Response Body should contain the ToDo we just posted to the server. Kitura is accessing a database running in a Kubernetes Cluster, while itself is running in the same cluster!

## Cleaning up

We now have a few things locally that are taking up disk space, including about 2.4GB of Docker images as well as a running cluster which is using system resources. Delete both the database and the kitura server from the cluster.

```bash
helm delete --purge server && helm delete --purge postgresql-database
```

We can now delete our local Docker images from the system to reclaim the lost hard drive space.

```bash
docker image list #We will need the IMAGE ID values
docker image rm -f IMAGE-ID #Repeat on the IMAGE ID of both your run and build images
```

We have now stopped the releases running in the Kubernetes cluster, and deleted their respective Docker images from the system.

### Tips for Kubernetes and Helm

For logs on the running Pods, use

```
kubectl log --timestamp=true --follow=true <POD-NAME>
```

This will create a streaming output of the logs created by the pod. This works well if you are having issues connecting to the database, for example. You could have logs from the database streaming as you accessed the todoserver on the port-forwarded port 8080, and get realtime feedback on what the server is reporting.

To delete an individual deployment created with Helm

```bash
helm list #Note the exact name (same one you chose when you ran helm install)
helm delete --purge HELM-NAME
```

Without including `--purge`, the name of the instance is not freed and if you ran `helm install` again using the same name, you could recieve an error.

You can also abbreviate `helm delete` to `helm del` .
