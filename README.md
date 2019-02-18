# Build a Continuous Deployment Multibranch Pipeline with Jenkins on GCP Kubernetes Engine

## Tutorial

[Docker : Continuous Delivery with Jenkins Multibranch Pipeline for Dev, Canary, and Production Environments on GCP Kubernetes Engine](https://bogotobogo.com/DevOps/Docker/Docker-Continuous-Delivery-with-Jenkins-Multibranch-Pipeline-for-Dev-Canary-Production-Environments-GCP-Kubernetes-Engine-Namespace.php)


## Introduction

In order to accomplish this goal we will use the following Jenkins plugins:
  - [Jenkins Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) - start Jenkins build executor containers in the Kubernetes cluster when builds are requested, terminate those containers when builds complete, freeing resources up for the rest of the cluster
  - [Jenkins Multibranch Pipelines](https://jenkins.io/solutions/pipeline/) - define our build pipeline declaratively and keep it checked into source code management alongside our application code
  - [Google Oauth Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Google+OAuth+Plugin) - allows us to add our google oauth credentials to jenkins

In order to deploy the application with [Kubernetes](http://kubernetes.io/) we will use the following resources:
  - [Deployments](http://kubernetes.io/docs/user-guide/deployments/) - replicates our application across our kubernetes nodes and allows us to do a controlled rolling update of our software across the fleet of application instances
  - [Services](http://kubernetes.io/docs/user-guide/services/) - load balancing and service discovery for our internal services
  - [Ingress](http://kubernetes.io/docs/user-guide/ingress/) - external load balancing and SSL termination for our external service
  - [Secrets](http://kubernetes.io/docs/user-guide/secrets/) - secure storage of non public configuration information, SSL certs specifically in our case

## Prerequisites
1. [Enable the Compute Engine, Container Engine, and Container Builder APIs](https://console.cloud.google.com/flows/enableapi?apiid=compute_component,container,cloudbuild.googleapis.com)

## Do this first

1. Create a new Google Cloud Platform project: [https://console.developers.google.com/project](https://console.developers.google.com/project)

1. Click the Google Cloud Shell icon in the top-right and wait for our shell to open:

1. When the shell is open, set default compute zone:

1. Clone the repository in our cloud shell, then `cd` into that dir:


## Create a Kubernetes Cluster
We'll use Google Container Engine to create and manage our Kubernetes cluster. Provision the cluster with `gcloud`:

Once that operation completes download the credentials for our cluster using the [gcloud CLI](https://cloud.google.com/sdk/):
```shell
$ gcloud container clusters get-credentials jenkins-cd
Fetching cluster endpoint and auth data.
kubeconfig entry generated for jenkins-cd.
```

Confirm that the cluster is running and `kubectl` is working by listing pods:

```shell
$ kubectl get pods
No resources found.
```
We should see `No resources found.`.

## Install Helm

We will use Helm to install Jenkins from the Charts repository. Helm is a package manager that makes it easy to configure and deploy Kubernetes applications.  Once we have Jenkins installed, we'll be able to set up our CI/CD pipleline.

1. Download and install the helm binary, and then unzip

    ```shell
    wget https://storage.googleapis.com/kubernetes-helm/helm-v2.9.1-linux-amd64.tar.gz

    tar zxfv helm-v2.9.1-linux-amd64.tar.gz

    cp linux-amd64/helm .
    ```

1. Add user as a cluster administrator in the cluster's RBAC so that we can give Jenkins permissions in the cluster:
    
    ```shell
    kubectl create clusterrolebinding cluster-admin-binding --clusterrole=cluster-admin --user=$(gcloud config get-value account)
    ```

1. Grant Tiller, the server side of Helm, the cluster-admin role in our cluster:

    ```shell
    kubectl create serviceaccount tiller --namespace kube-system
    kubectl create clusterrolebinding tiller-admin-binding --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
    ```

1. Initialize Helm. This ensures that the server side of Helm (Tiller) is properly installed in our cluster.

    ```shell
    ./helm init --service-account=tiller
    ./helm update
    ```

1. Ensure Helm is properly installed by running the following command. We should see versions appear for both the server and the client of ```v2.9.1```:

    ```shell
    ./helm version
    Client: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
    Server: &version.Version{SemVer:"v2.9.1", GitCommit:"20adb27c7c5868466912eebdf6664e7390ebe710", GitTreeState:"clean"}
    ```

## Configure and Install Jenkins
We will use a custom [values file](https://github.com/kubernetes/helm/blob/master/docs/chart_template_guide/values_files.md) to add the GCP specific plugin necessary to use service account credentials to reach our Cloud Source Repository.

1. Use the Helm CLI to deploy the chart with our configuration set.

    ```shell
    ./helm install -n cd stable/jenkins -f jenkins/values.yaml --version 0.16.6 --wait
    ```

1. Once that command completes ensure the Jenkins pod goes to the `Running` state and the container is in the `READY` state:

    ```shell
    $ kubectl get pods
    NAME                          READY     STATUS    RESTARTS   AGE
    cd-jenkins-7c786475dd-vbhg4   1/1       Running   0          1m
    ```

1. Run the following command to setup port forwarding to the Jenkins UI from the Cloud Shell

    ```shell
    export POD_NAME=$(kubectl get pods -l "component=cd-jenkins-master" -o jsonpath="{.items[0].metadata.name}")
    kubectl port-forward $POD_NAME 8080:8080 >> /dev/null &
    ```

1. Now, check that the Jenkins Service was created properly:

    ```shell
    $ kubectl get svc
    NAME               CLUSTER-IP     EXTERNAL-IP   PORT(S)     AGE
    cd-jenkins         10.35.249.67   <none>        8080/TCP    3h
    cd-jenkins-agent   10.35.248.1    <none>        50000/TCP   3h
    kubernetes         10.35.240.1    <none>        443/TCP     9h
    ```

We are using the [Kubernetes Plugin](https://wiki.jenkins-ci.org/display/JENKINS/Kubernetes+Plugin) so that our builder nodes will be automatically launched as necessary when the Jenkins master requests them.
Upon completion of their work they will automatically be turned down and their resources added back to the clusters resource pool.

Notice that this service exposes ports `8080` and `50000` for any pods that match the `selector`. This will expose the Jenkins web UI and builder/agent registration ports within the Kubernetes cluster.
Additionally the `jenkins-ui` services is exposed using a ClusterIP so that it is not accessible from outside the cluster.

## Connect to Jenkins

1. The Jenkins chart will automatically create an admin password for us. To retrieve it, run:

    ```shell
    printf $(kubectl get secret cd-jenkins -o jsonpath="{.data.jenkins-admin-password}" | base64 --decode);echo
    ```

2. To get to the Jenkins user interface, click on the Web Preview button![](../docs/img/web-preview.png) in cloud shell, then click “Preview on port 8080”:

![](docs/img/preview-8080.png)

We should now be able to log in with username `admin` and our auto generated password.

![](docs/img/jenkins-login.png)

### Our progress, and what's next
We've got a Kubernetes cluster managed by Google Container Engine. We've deployed:

* a Jenkins Deployment
* a (non-public) service that exposes Jenkins to its agent containers

We have the tools to build a continuous deployment pipeline. Now we need a sample GO app to deploy continuously.

## The sample GO app
We'll use a very simple sample application written in GO - `gceme` - as the basis for our CD pipeline. `gceme` is written in Go and is located in the `sample-app` directory in this repo. When we run the `gceme` binary on a GCE instance, it displays the instance's metadata in a pretty card:

![](docs/img/info_card.png)

The binary supports two modes of operation, designed to mimic a microservice. In backend mode, `gceme` will listen on a port (8080 by default) and return GCE instance metadata as JSON, with content-type=application/json. In frontend mode, `gceme` will query a backend `gceme` service and render that JSON in the UI we saw above. It looks roughly like this:

```
-----------      ------------      ~~~~~~~~~~~~        -----------
|         |      |          |      |          |        |         |
|  user   | ---> |   gceme  | ---> | lb/proxy | -----> |  gceme  |
|(browser)|      |(frontend)|      |(optional)|   |    |(backend)|
|         |      |          |      |          |   |    |         |
-----------      ------------      ~~~~~~~~~~~~   |    -----------
                                                  |    -----------
                                                  |    |         |
                                                  |--> |  gceme  |
                                                       |(backend)|
                                                       |         |
                                                       -----------
```
Both the frontend and backend modes of the application support two additional URLs:

1. `/version` prints the version of the binary (declared as a const in `main.go`)
1. `/healthz` reports the health of the application. In frontend mode, health will be OK if the backend is reachable.

### Deploy the sample GO app to Kubernetes
In this section we will deploy the `gceme` frontend and backend to Kubernetes using Kubernetes manifest files (included in this repo) that describe the environment that the `gceme` binary/Docker image will be deployed to. They use a default `gceme` Docker image that we will be updating with our own in a later section.

We'll have two primary environments - [canary](http://martinfowler.com/bliki/CanaryRelease.html) and production - and use Kubernetes to manage them.

> **Note**: The manifest files for this section of the tutorial are in `sample-app/k8s`. We are encouraged to open and read each one before creating it per the instructions.

1. First change directories to the sample-app:

  ```shell
  $ cd sample-app
  ```

1. Create the namespace for production:

  ```shell
  $ kubectl create ns production
  ```

1. Create the canary and production Deployments and Services:

    ```shell
    $ kubectl --namespace=production apply -f k8s/production
    $ kubectl --namespace=production apply -f k8s/canary
    $ kubectl --namespace=production apply -f k8s/services
    ```

1. Scale the production service:

    ```shell
    $ kubectl --namespace=production scale deployment gceme-frontend-production --replicas=4
    ```

1. Retrieve the External IP for the production services: **This field may take a few minutes to appear as the load balancer is being provisioned**:

  ```shell
  $ kubectl --namespace=production get service gceme-frontend
  NAME             TYPE           CLUSTER-IP     EXTERNAL-IP    PORT(S)        AGE
  gceme-frontend   LoadBalancer   10.35.254.91   35.196.48.78   80:31088/TCP   1m
  ```

1. Confirm that both services are working by opening the frontend external IP in our browser

1. Open a new Google Cloud Shell terminal by clicking the `+` button to the right of the current terminal's tab, and poll the production endpoint's `/version` URL. Leave this running in the second terminal so we can easily observe rolling updates in the next section:

   ```shell
   $ export FRONTEND_SERVICE_IP=$(kubectl get -o jsonpath="{.status.loadBalancer.ingress[0].ip}"  --namespace=production services gceme-frontend)
   $ while true; do curl http://$FRONTEND_SERVICE_IP/version; sleep 1;  done
   ```

1. Return to the first terminal

### Create a repository for the sample GO app source
Here we'll create our own copy of the `gceme` sample GO app in [Cloud Source Repository](https://cloud.google.com/source-repositories/docs/).

1. Change directories to `sample-app` of the repo we cloned previously, then initialize the git repository.

   **Be sure to replace _REPLACE_WITH_OUR_PROJECT_ID_ with the name of our Google Cloud Platform project**

    ```shell
    $ cd sample-app
    $ git init
    $ git config credential.helper gcloud.sh
    $ gcloud source repos create gceme
    $ git remote add origin https://source.developers.google.com/p/REPLACE_WITH_OUR_PROJECT_ID/r/gceme
    ```

1. Ensure git is able to identify us:

    ```shell
    $ git config --global user.email "OUR-EMAIL-ADDRESS"
    $ git config --global user.name "OUR-NAME"
    ```

1. Add, commit, and push all the files:

    ```shell
    $ git add .
    $ git commit -m "Initial commit"
    $ git push origin master
    ```

## Create a pipeline
We'll now use Jenkins to define and run a pipeline that will test, build, and deploy our copy of `gceme` to our Kubernetes cluster. We'll approach this in phases. Let's get started with the first.

### Phase 1: Add our service account credentials
First we will need to configure our GCP credentials in order for Jenkins to be able to access our code repository

1. In the Jenkins UI, Click “Credentials” on the left
1. Click either of the “(global)” links (they both route to the same URL)
1. Click “Add Credentials” on the left
1. From the “Kind” dropdown, select “Google Service Account from metadata”
1. Click “OK”

We should now see 2 Global Credentials. Make a note of the name of second credentials as we will reference this in Phase 2:

![](docs/img/jenkins-credentials.png)


### Phase 2: Create a job
This lab uses [Jenkins Pipeline](https://jenkins.io/solutions/pipeline/) to define builds as groovy scripts.

Navigate to our Jenkins UI and follow these steps to configure a Pipeline job (hot tip: we can find the IP address of our Jenkins install with `kubectl get ingress --namespace jenkins`):

1. Click the “Jenkins” link in the top left of the interface

1. Click the **New Item** link in the left nav

1. Name the project **sample-G0-app**, choose the **Multibranch Pipeline** option, then click `OK`

1. Click `Add Source` and choose `git`

1. Paste the **HTTPS clone URL** of our `sample-app` repo on Cloud Source Repositories into the **Project Repository** field.
    It will look like: https://source.developers.google.com/p/REPLACE_WITH_OUR_PROJECT_ID/r/gceme

1. From the Credentials dropdown select the name of new created credentials from the Phase 1. It should have the format `PROJECT_ID service account`.

1. Under 'Scan Multibranch Pipeline Triggers' section, check the 'Periodically if not otherwise run' box and se the 'Interval' value to 1 minute.

1. Click `Save`, leaving all other options with their defaults

  ![](docs/img/clone_url.png)

A job entitled "Branch indexing" was kicked off to see identify the branches in our repository. If we refresh Jenkins we should see the `master` branch now has a job created for it.

The first run of the job will fail until the project name is set properly in the next step.

### Phase 3:  Modify Jenkinsfile, then build and test the app

Create a branch for the canary environment called `canary`
   
   ```shell
    $ git checkout -b canary
   ```

The [`Jenkinsfile`](https://jenkins.io/doc/book/pipeline/jenkinsfile/) is written using the Jenkins Workflow DSL (Groovy-based). It allows an entire build pipeline to be expressed in a single script that lives alongside our source code and supports powerful features like parallelization, stages, and user input.

Modify our `Jenkinsfile` script so it contains the correct project name on line 2.

**Be sure to replace _REPLACE_WITH_OUR_PROJECT_ID_ on line 2 with our project name:**

Don't commit the new `Jenkinsfile` just yet. We'll make one more change in the next section, then commit and push them together.

### Phase 4: Deploy a [canary release](http://martinfowler.com/bliki/CanaryRelease.html) to canary
Now that our pipeline is working, it's time to make a change to the `gceme` app and let our pipeline test, package, and deploy it.

The canary environment is rolled out as a percentage of the pods behind the production load balancer.
In this case we have 1 out of 5 of our frontends running the canary code and the other 4 running the production code. This allows we to ensure that the canary code is not negatively affecting users before rolling out to our full fleet.
We can use the [labels](http://kubernetes.io/docs/user-guide/labels/) `env: production` and `env: canary` in Google Cloud Monitoring in order to monitor the performance of each version individually.

1. In the `sample-app` repository on our workstation open `html.go` and replace the word `blue` with `orange` (there should be exactly two occurrences):

  ```html
  //snip
  <div class="card orange">
  <div class="card-content white-text">
  <div class="card-title">Backend that serviced this request</div>
  //snip
  ```

1. In the same repository, open `main.go` and change the version number from `1.0.0` to `2.0.0`:

   ```go
   //snip
   const version string = "2.0.0"
   //snip
   ```

1. `git add Jenkinsfile html.go main.go`, then `git commit -m "Version 2"`, and finally `git push origin canary` our change.

1. When our change has been pushed to the Git repository, navigate to our Jenkins job. Click the "Scan Multibranch Pipeline Now" button.

  ![](docs/img/first-build.png)

1. Once the build is running, click the down arrow next to the build in the left column and choose **Console Output**:

  ![](docs/img/console.png)

1. Track the output for a few minutes and watch for the `kubectl --namespace=production apply...` to begin. When it starts, open the terminal that's polling canary's `/version` URL and observe it start to change in some of the requests:

   ```
   1.0.0
   1.0.0
   1.0.0
   1.0.0
   2.0.0
   2.0.0
   1.0.0
   1.0.0
   1.0.0
   1.0.0
   ```

   We have now rolled out that change to a subset of users.

1. Once the change is deployed to canary, we can continue to roll it out to the rest of our users by creating a branch called `production` and pushing it to the Git server:

   ```shell
    $ git checkout master
    $ git merge canary
    $ git push origin master
   ```
1. In a minute or so we should see that the master job in the sample-app folder has been kicked off:

    ![](docs/img/production.png)

1. Clicking on the `master` link will show we the stages of our pipeline as well as pass/fail and timing characteristics.

    ![](docs/img/production_pipeline.png)

1. Open the terminal that's polling canary's `/version` URL and observe that the new version (2.0.0) has been rolled out and is serving all requests.

   ```
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   2.0.0
   ```

1. Look at the `Jenkinsfile` in the project to see how the workflow is written.

### Phase 5: Deploy a development branch
Often times changes will not be so trivial that they can be pushed directly to the canary environment. In order to create a development environment from a long lived feature branch
all we need to do is push it up to the Git server and let Jenkins deploy our environment. In this case we will not use a loadbalancer so we'll have to access our application using `kubectl proxy`,
which authenticates itself with the Kubernetes API and proxies requests from our local machine to the service in the cluster without exposing our service to the internet.

#### Deploy the development branch

1. Create another branch and push it up to the Git server

   ```shell
   $ git checkout -b new-feature
   $ git push origin new-feature
   ```

1. Open Jenkins in our web browser and navigate to the sample-app job. We should see that a new job called "new-feature" has been created and our environment is being created.

1. Navigate to the console output of the first build of this new job by:

  * Click the `new-feature` link in the job list.
  * Click the `#1` link in the Build History list on the left of the page.
  * Finally click the `Console Output` link in the left navigation.

1. Scroll to the bottom of the console output of the job, and we will see instructions for accessing our environment:

   ```
   deployment "gceme-frontend-dev" created
   [Pipeline] echo
   To access our environment run `kubectl proxy`
   [Pipeline] echo
   Then access our service via http://localhost:8001/api/v1/proxy/namespaces/new-feature/services/gceme-frontend:80/
   [Pipeline] }
   ```

#### Access the development branch

1. Open a new Google Cloud Shell terminal by clicking the `+` button to the right of the current terminal's tab, and start the proxy:

   ```shell
   $ kubectl proxy
   ```

1. Return to the original shell, and access our application via localhost:

   ```shell
   $ curl http://localhost:8001/api/v1/proxy/namespaces/new-feature/services/gceme-frontend:80/
   ```

1. We can now push code to the `new-feature` branch in order to update our development environment.

1. Once we are done, merge our `new-feature ` branch back into the  `canary` branch to deploy that code to the canary environment:

   ```shell
   $ git checkout canary
   $ git merge new-feature
   $ git push origin canary
   ```

1. When we are confident that our code won't wreak havoc in production, merge from the `canary` branch to the `master` branch. Our code will be automatically rolled out in the production environment:

   ```shell
   $ git checkout master
   $ git merge canary
   $ git push origin master
   ```

1. When we are done with our development branch, delete it from the server and delete the environment in Kubernetes:

   ```shell
   $ git push origin :new-feature
   $ kubectl delete ns new-feature
   ```

## Extra credit: deploy a breaking change, then roll back
Make a breaking change to the `gceme` source, push it, and deploy it through the pipeline to production. Then pretend latency spiked after the deployment and we want to roll back. Do it! Faster!

Things to consider:

* What is the Docker image we want to deploy for roll back?
* How can we interact directly with the Kubernetes to trigger the deployment?
* Is SRE really what we want to do with our life?

## Clean up
Clean up is really easy, but also super important: if we don't follow these instructions, we will continue to be billed for the Google Container Engine cluster we created.

To clean up, navigate to the [Google Developers Console Project List](https://console.developers.google.com/project), choose the project we created for this lab, and delete it. That's it.

## Credit
[Continuous Delivery with Jenkins in Kubernetes Engine](https://github.com/GoogleCloudPlatform/continuous-deployment-on-kubernetes.git)
