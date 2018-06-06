# Building a "To Do" Backend with Kitura

<p align="center">
<img src="https://www.ibm.com/cloud-computing/bluemix/sites/default/files/assets/page/catalog-swift.svg" width="120" alt="Kitura Bird">
</p>

<p align="center">
<a href= "http://swift-at-ibm-slack.mybluemix.net/"> 
    <img src="http://swift-at-ibm-slack.mybluemix.net/badge.svg"  alt="Slack"> 
</a>
</p>

In this self-paced tutorial you will build a Kitura 2 application from scratch and create a REST API inside the application.

The application you'll create is a "ToDo list" application, as described by the [Todo-Backend](http://todobackend.com/) project.

You'll learn about server-side Swift, the Kitura framework, REST APIs, and OpenAPI.

At the end you should have a fully functioning application which passes a provided verification testsuite.

## Prerequisites

Before getting started, make sure you have the following prerequisites installed on your system.

1. macOS
2. Xcode 9.3 or later. You can install Xcode from the Mac App Store.
3. Command line tools for Xcode. You can check if these are installed (and install them if necessary) by running `xcode-select --install` in a terminal window.

## Setting up

Start by cloning this repository to your system:

```
cd ~
git clone https://github.com/IBM/ToDoBackend.git
```

Then, clone the testsuite you'll be using to verify your REST API:

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

The Kitura macOS app provides an easy point-and-click way to generate a new Kitura project. There are two templates: Skeleton and Starter.

1. Mouse over "Starter" and click "Create".
2. Navigate to the "ToDoBackend" folder in your home folder.
3. Change the project name to "ToDoServer".
3. Click "Create".

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

Congratulations, you have created your first Kitura application. Proceed to [the rest of the workshop](https://github.com/IBM/ToDoBackend/blob/master/Workshop.md).

### Option 3: Create your Kitura starter kit in IBM Cloud

1. Start by visiting [IBM Cloud](https://console.bluemix.net) in your web browser.
2. If you already have a free IBM Cloud account, skip to the next step. Otherwise, click "Create a free Account", complete the form, and click "Create Account". Confirm your account by accessing your email and clicking "Confirm Account" in the email you are sent.
3. Log in to IBM Cloud with your credentials.

A starter kit is a pre-configured application template that allows you to deploy a production-ready application to IBM Cloud within minutes.

Code generation technology creates an application in your preferred language and framework, which can be tailored to your needs and use case. Any services that are required in support of the use case are provisioned automatically.

You can debug and test on your local workstation or in the cloud, and then
use a DevOps toolchain to automate the delivery process.

1. In the IBM Cloud dashboard, click the top left "hamburger" icon to open the navigation menu, then choose "Apple Development". This opens the IBM Apple Developer Service.
2. Click "Starter Kits" in the left menu.
3. Click "Create App".
4. Name your project "ToDoServer".
5. Under "Select your language" choose "Swift" so that a server-side Swift app is created.
6. Click the "Create" button on the right hand side to create your app.

The app details view is now displayed where you could configure your app, add services, and more.

1. Click the "Download Code" button on the top right to download your Kitura starter kit application in a zip file.

Your browser may automatically unzip the file for you. If it does not, unzip your app and copy it to the workshop location as follows:

```
cd ~/Downloads
mkdir ToDoServer
unzip -d ToDoServer <filename>.zip  # <filename>.zip was downloaded from IBM Cloud
cp -R ToDoServer ~/ToDoBackend/ToDoServer
```

If it is already unzipped, just copy the files to the workshop location:

```
cd ~/Downloads
cp -R <foldername> ~/ToDoBackend/ToDoServer
```

Now change to your project folder and generate an Xcode project for the rest of the workshop:

```
cd ~/ToDoBackend/ToDoServer
swift package generate-xcodeproj
```

Congratulations, you have created your first Kitura application. Proceed to [the rest of the workshop](https://github.com/IBM/ToDoBackend/blob/master/Workshop.md).
