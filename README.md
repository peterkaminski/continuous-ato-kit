# Continuous ATO Kit

Incorporating Compliance as Code and continuous compliance within the proof-of-concept CI/CD pipeline.

![Diagram showing a build pipeline environment consisting of a Source Code Repository (GitHub) a Build Server (Jenkins), a Target Application Instance, a Security and Monitoring Server (OpenSCAP), a Compliance Server (GovReady-Q), a DevSecOps Workstation, and Production Environment.](docs/c-a-k-system-diagram-p1.png)

1. [Overview](#overview)
1. [Step 1 - Install Docker](#docker)
1. [Step 2 - Get the Kit](#getkit)
1. [Step 3 - Set Up the Pipeline Environment](#pipeline)
	1. [Step 3(a) - Start the Servers and Network](#network)
	1. [Step 3(b) - Set Up Build Server](#build_server)
	1. [Step 3(c) - Set Up Security and Monitoring Server](#security_server)
	1. [Step 3(d) - Set Up Compliance Server](#compliance_server)
1. [Step 4 - Set Up Build Task](#buildtask)
1. [Step 5 - Build Target App](#build)
1. [Step 6 - View Compliance Artifacts](#view)
1. [Discussion](#discussion)
1. [Tear-down](#deployment)

# <a name="overview"></a> Overview

This proof-of-concept demonstrates how security checks, compliance testing, and compliance artifact maintenance can be just another automated part of the CI/CD pipeline.

Once set up, the pipeline's Build Server (Jenkins) will assemble a Target App, tell a Security Server (OpenSCAP) to scan the Target App and its OS, then tell a Compliance Server (GovReady) to check compliance and update a System Security Plan.

![sequence](docs/awesome-ato-sequence.png)

The pipeline is simplifed to run on a single workstation playing the role of both Pipeline Environment (Docker Host) and DevSecOps Workstations. Instructions refer to the Pipeline Environment and DevSecOps Workstation separately in order to communicate what is happening functionally, even though they are implemented by the same workstation in this demonstration.

# <a name="docker"></a> Step 1 - Install Docker

First [install Docker](https://docs.docker.com/engine/installation/) and Docker Compose on the workstation that will provide the Docker Host Pipeline Environment.

* On Mac and Windows, Docker Compose is included as part of the Docker install.

* On Linux, after installing Docker, [install Docker Compose](https://docs.docker.com/compose/install/#install-compose). Also, you may want to [grant non-root users access to run Docker containers](https://docs.docker.com/engine/installation/linux/linux-postinstall/#manage-docker-as-a-non-root-user).

# <a name="getkit"></a> Step 2 - Get the Kit

Get the Continuous ATO Kit by cloning this repository onto the Docker Host Pipeline Environment.

	git clone https://github.com/GovReady/continuous-ato-kit
	cd continuous-ato-kit

# <a name="pipeline"></a> Step 3 - Set Up the Pipeline Environment

## <a name="network"></a> Step 3(a) - Start the Servers and Network

Start the Build Server, the Security and Monitoring Server, and the Compliance Server:

	./atokit-up.sh

This script uses Docker Compose to start the servers.  You will see the Docker build steps, and then the output will pause while the applications start up.

![servers up in Terminal](docs/servers-up.png)

* When the **Build Server** is up, you will see the message *"INFO: Jenkins is fully up and running"*.
* When the **Security and Monitoring Server** is up, you will see the message *"INFO: Security and Monitoring Server is fully up and running."*
* When the **Compliance Server** is up, you will see the message *"GovReady-Q is fully up and running."*

When all three servers are up, exit the `atokit-up.sh` by hitting control-C.  The log viewer will exit with the message "ERROR: Aborting", which is expected, not an error.  The servers will continue to run in the background, communicating with each other using a Docker user-defined network called `continuousatokit_ato_network` -- a private virtual network running entirely on your computer.

You can verify the Docker containers are up by running `docker-compose ps` in a terminal on the Docker Host.

Your Pipeline Environment now exists. In the next step, we'll set up and configure the software in our pipeline.

## <a name="build_server"></a> Step 3(b) - Set Up Build Server

The Build Server is Jenkins. In this step, we will unlock Jenkins, install plugins, and create an administration account.

In your terminal, run the following script to display the Jenkins unlock password created when Jenkins was spun up:

    ./get-jenkins-password.sh

Select and copy the displayed password.

Open a browser and navigate to the Jenkins administration interface at:

	http://localhost:8080/

You should see a page named Unlock Jenkins. Paste the unlock code from the terminal into Jenkins to login.

Choose “Install Suggested Packages”, then “Continue as Admin”, then “Start Using Jenkins”.

* See the [Jenkins documentation](https://jenkins.io/doc/tutorials/building-a-node-js-and-react-app-with-npm/) for further information about starting Jenkins.

Your Jenkins Build Server is now set up. Next, we'll set up the Security and Monitoring Server.

## <a name="security_server"></a> Step 3(b) - Set Up Security and Monitoring Server

The Security and Monitoring Server is based on a CentOS 7 image and is automatically set up with OpenSCAP and ready to run during the first build process. So there's nothing specific to do for this step.

Next, we'll set up the Compliance Server.

## <a name="compliance_server"></a> Step 3(c) -  Set Up Compliance Server

The Compliance Server is GovReady-Q. In this step, we will add an alias entry for GovReady-Q to our host file, create an administration account, install the correct compliance app to display the compliance of our target app, and share credentials with Jenkins.

Add an alias entry for GovReady-Q to the `/etc/hosts` file on the workstation from which you will view GovReady-Q web pages.

	127.0.0.1	govready-q

(ADVANCED CONFIGURATION NOTE: Use the IP address of the Docker Host if the Docker Host is different from your workstation.)

Your OpenSCAP Security and Monitoring Server is now set up. Next, we'll set up the Compliance Server.

Now open the GovReady-Q Compliance Server in a web browser at:

	http://govready-q:8000


Next, return to a terminal on the Docker Host and run the following command:

	docker-compose exec govready-q ./first_run.py

A new administrative user will be created on the GovReady-Q Compliance Server. The username and password will be written to the console:

	Creating administrative user...
	Username: demo
	Password: 9kAxaPW6hJVLsscf5jbWn6vc
	API Key: bdAq16aGh0ybzMAWMioCyWqpb2wItlYo
	Creating default organization...
	Adding TACR Demo Apps...
	Starting compliance app...
	...
	API URL: http://govready-q:8000/api/v1/organizations/main/projects/3/answers

You can now log into GovReady-Q and view your compliance app.

We can now provide our Jenkins Build Server with credentials to talk directly to our GovReady-Q Compliance Server.

In Jenkins, go to the top level of Jenkins, and then to the Credentials page.

Click on a credential scope, such as the global scope. Click on “Add credentials”. Change “Kind” to “Secret text”. For the “Secret”, paste the GovReady-Q Unix Server API URL found in the console output from earlier. For “ID”, enter `govready_q_api_url`. Optionally add a description. And click “OK”.

![](jenkins-credentials-2.png)

Add a second “Secret text” credential in the same manner where the “Secret” is the GovReady-Q API Key found in the console output earlier and the “ID” is `govready_q_api_key`.

Once the credentials have been set, they will look like this:

![](jenkins-credentials-1.png)

Add a third credential whose kind is “Secret file”. Browse to [security-server/keys/id_ecdsa.pub](security-server/keys/id_ecdsa.pub) in this repository to select it. Set the credential ID to `target_ecdsa.pub`.

Your GovReady-Q Compliance Server is now set up.

In the next step, we will set up the build task in Jenkins Build Server. Then we will be ready to run our build and watch our Security and Monitoring Server scan our target app and update our Compliance Artifacts in our GovReady-Q Compliance Server.

# <a name="buildtask"></a> Step 4 - Set up Build Task

For the purposes of this demo, we will build the Jenkinsfile in this repository. We will have Jenkins pull the code in this repository directly from GitHub. (Or if you prefer, you can clone the repository into your GitHub account or another git host, and use that one, or use the Advanced Jenkins configuration earlier to load it from the local disk.)

* Start at the Jenkins dashboard, at http://localhost:8080/

* Click on “New Item”.

* Enter an item name, such as “Continuous ATO Kit”.

* Click “Pipeline” as the type of project, then click “OK” at the bottom of the screen.

* Now, to configure the project, click the “Pipeline” tab to scroll down to the Pipeline section.

* For “Definition”, choose “Pipeline script from SCM”.  This will tell Jenkins to look in a repository for a Jenkinsfile to use as the pipeline script.

* For “SCM”, choose “Git”.  Then for “Repository URL”, enter the URL for the repository, which is `https://github.com/GovReady/continuous-ato-kit`.

* You can leave “Credentials” set to “none”.  (*Advanced*. For a private repository, you could set up a GitHub personal access token for Jenkins to use, and then provide it to Jenkins here.)

* Click “Save”, and you’re ready to build.


# <a name="build"></a> Step 5 - Build Target App

Return to the top level of Jenkins, then in the list of projects open the Continuous ATO Kit pipeline project.

Click “Open in Blue Ocean”. Then click “Run”. When the build appears in the History...

TODO: Explain how to build.


# <a name="view"></a> Step 6 - View Compliance Artifacts

Now view the results in the GovReady-Q Compliance Server. Using the username and password for the GovReady-Q Compliance Server output on the console, log into GovReady-Q `http://govready-q:8000`.

Go to “App Folder”, then “TACR SSP All”.

Click “Review” to inspect the information uploaded to the compliance server during the build.

Go back, then click “SSP Preview” at the bottom of the page. Scroll down to CM-6: Configuration Settings and check that the end of the section reads:

 A security scan was performed on the system (hostname 02a425ff11e7). The report can be downloaded here.

# <a name="discussion"></a> Discussion

The target application uses a `Jenkinsfile` to define the build, test, and deployment steps for the application. The `Jenkinsfile` is stored in the **Application Source Code Repository** and will be fetched by Jenkins when a build is initiated.

Some of the important parts of the Jenkins file are described below.

	pipeline {
	  agent {
	    docker {
	      image 'python:3'
	      args '--network continuousatokit_ato_network'
	    }
	  }

The `agent` section defines the properties of an ephemeral Docker container that runs the build steps. This is also our **Target Application Server**. The container is added to the virtual network created earlier using `--network continuousatokit_ato_network`, which allows the build container to communicate with the **Compliance Server**.

	stages {
	stage('Build App') {
	  steps {
	    sh 'pip install -r requirements.txt'
	  }
	}

The build phase contains typical Jenkins build instructions. In this case, we install the target application’s Python package dependencies.

    stage('Test App') {
      steps {
        sh './manage.py test 2>&1 | tee /tmp/pytestresults.txt'

The test stage runs the target application’s tests (unit tests, integration tests, etc.). The Unix `tee` command is used to save the test results to a temporary file while also retaining the console output that is fed to Jenkins.

	        withCredentials([
	        	string(credentialsId: 'govready_q_api_url', variable: 'Q_API_URL'),
	        	string(credentialsId: 'govready_q_api_key', variable: 'Q_API_KEY')]) {

	            // Send hostname.
	            sh 'curl -F project.file_server.hostname=$(hostname)
	                     --header "Authorization:$Q_API_KEY" $Q_API_URL'

	            // Send test results.
	            sh 'curl -F "project.file_server.login_message=</tmp/pytestresults.txt"
	                     --header "Authorization:$Q_API_KEY" $Q_API_URL'
	        }
	      }
	    }
	  }
	}

The test stage then sends information about the build to the **Compliance Server** using the GovReady-Q Compliance App’s API. `withCredentials` is a Jenkins command that pulls credentials (see above) into environment variables. Within `withCredentials`, two API calls are made:

1. The first API call sends the build machine’s hostname, as returned by the Unix `hostname` command, and stores it in the Compliance App’s `project.file_server.hostname` data field.

2. The second API call sends the saved test results from the temporary file and stores it in the Compliance App’s `project.file_server.login_message` data field. (TODO: Change the field name.)

#### Further Steps (TODO)

* Start the build.
* Watch Jenkins run the build tasks.
* Watch Jenkins run the testing tasks of the application and the security and monitoring server.
* Watch Jenkins share information with the compliance server to update the System Security Plan and ATO.


# <a name="tear"></a> Tear-down

Remove the containers, the network, and the persistent data volume for Jenkins started by docker-compose:

	./atokit-down.sh

There are official base images for GovReady-Q, Jenkins, and CentOS 7 that were pulled down to support the builds, which you may want to keep installed within Docker.  Or if you prefer to remove them to save disk space, you can run:

	./atokit-rm.sh
