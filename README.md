# Building a "ToDo" Backend with Kitura

<p align="center">
<img src="https://www.ibm.com/cloud-computing/bluemix/sites/default/files/assets/page/catalog-swift.svg" width="120" alt="Kitura Bird">
</p>

<p align="center">
<a href= "http://swift-at-ibm-slack.mybluemix.net/">
    <img src="http://swift-at-ibm-slack.mybluemix.net/badge.svg"  alt="Slack">
</a>
</p>

## Workshop Table of Contents:

1. **[Build your Kitura app](https://github.com/IBM/ToDoBackend/blob/master/README.md)**
2. [Connect it to an SQL database](https://github.com/IBM/ToDoBackend/blob/master/DatabaseWorkshop.md)
3. [Build your app into a Docker image and deploy it on Kubernetes.](https://github.com/IBM/ToDoBackend/blob/master/DeployingToKube.md)
4. [Enable monitoring through Prometheus/Grafana](https://github.com/IBM/ToDoBackend/blob/master/MonitoringKube.md)

# Building your Kitura app

The application you'll create is a "ToDo list" application, as described by the [Todo-Backend](http://todobackend.com/) project.

You'll learn about server-side Swift, the Kitura framework, REST APIs, OpenAPI, Docker and Kubernetes.

At the end you should have a fully functioning application which passes a provided verification testsuite, running in Kubernetes.

## Prerequisites

Before getting started, make sure you have the following prerequisites installed on your system.

1. macOS
2. Xcode 9.3 or later. You can install Xcode from the Mac App Store.
3. Command line tools for Xcode. You can check if these are installed (and install them if necessary) by running `xcode-select --install` in a terminal window.

## Setting up

Start by cloning the testsuite to your system. You'll be using this to verify your REST API:

```
cd ~
git clone https://github.com/TodoBackend/todo-backend-js-spec.git
```

The next step is to generate your first Kitura application. Using an application generator will get you started quickly and ensure that your Kitura application is cloud-ready.

There are three ways you can generate your application:

1. Using the Kitura macOS app.
2. At the command-line, by installing the Kitura command-line interface via Homebrew.
3. Using a web browser to create a Kitura "starter kit" in IBM Cloud, then downloading the generated code in a zip file. This requires signing up for a free IBM Cloud account (no credit card is required).

You can choose whichever option you prefer.

### Option 1: Use the Kitura macOS app

1. Visit [https://www.kitura.io/app.html](https://www.kitura.io/app.html) in your web browser and download the Kitura app.
2. Install the app by opening the downloaded `Kitura.dmg` and dragging the app to your `Applications` folder.
3. Ctrl-click the `Kitura` app in your `Applications` folder and choose "Open".

The Kitura macOS app provides an easy point-and-click way to generate a new Kitura project. There are three templates: Skeleton, Starter, and OpenAPI.

1. Mouse over "OpenAPI" and click "Create".
2. Navigate to the "ToDoBackend" folder in your home folder.
3. Change the project name to "ToDoServer".
4. Click "Create".

The Kitura app will create a new Kitura project for you, ready to deploy to the cloud.

Congratulations, you have created your first Kitura application. Proceed to [the rest of the workshop](https://github.com/IBM/ToDoBackend/blob/master/Workshop.md).

### Option 2: Create your Kitura application at the command-line

1. Open a terminal window and check that you have Homebrew installed by running `brew --version`. If you need to install Homebrew, visit [brew.sh](https://brew.sh/) and follow the installation instructions.
2. Install the Kitura command-line interface:  
   1. Add the Kitura tap to your Homebrew: `brew tap ibm-swift/kitura`  
   2. Install the Kitura CLI: `brew install kitura`

You can check that the Kitura CLI has been installed correctly by running `kitura --help`.

Now, generate your Kitura application:

```
mkdir ~/ToDoBackend/ToDoServer
cd ~/ToDoBackend/ToDoServer
kitura init
```

This creates a fully working Kitura project that provides monitoring and metrics which can then be extended with your application logic.

Congratulations, you have created your first Kitura application.  Proceed to [the rest of the workshop](https://github.com/IBM/ToDoBackend/blob/master/Workshop.md).
