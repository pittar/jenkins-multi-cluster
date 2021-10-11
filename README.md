# Jenkins Multi-Cluster Access

This is a simplified pipeline adapted from [Gerald Nunn's example](https://github.com/gnunn1/openshift-basic-pipeline/tree/master/openshift/cross-cluster).

## Setting Up "Production" Cluster

The "production" cluster will not have Jenkins installed on it, but Jenkins running in "non-prod" will need access to this cluster.  For this, we will need a service account.

Login to the "prod" cluster with the `oc` command line tool, create a new "cicd" project (that will hold the new service account), and create the Jenkins service account:

```
oc new-project cicd

oc create sa jenkins -n cicd
```

You will now have a `jenkins` service account in the `cicd` project in your production cluster.

Next, you will have to give this service account access to the project(s) in your production cluster that you want Jenkins to be able to manage.  For example, if you want Jenkins to manage the **myapp** project:

```
oc adm policy add-role-to-user admin system:serviceaccount:cicd:jenkins -n myapp-preprod
oc adm policy add-role-to-user admin system:serviceaccount:cicd:jenkins -n myapp-prod
```

The Jenkins service account now has `admin` access to the `myapp` project.  Repeat the command above for any other projects that you want Jenkins to manage.

Finally, you will need to get the access token for this service account (to be used by Jenkins to login to this cluster), as well as the production server api url.

```
# Get the PROD server API URL
oc whoami --show-server

# Get the token of the Jenkins service account (it will be very long - be sure to copy it carefully)
oc sa get-token jenkins -n cicd
```

Now, configure Jenkins so it can connect to the "prod" cluster.

1. Login to the Jenkins UI.
2. Navigate to **Manage Jenkins -> Configure System**
3. Scroll down to the "OpenShift Client Plugin" section and click select **Add OpenShift Cluster**
4. Enter the following values in the fields that appear:
    * Cluster Name: `production`
    * API Server URL: `<the url:port output from "oc whoami --show-server">`
    * Credentials: Click "Add -> Jenkins"
        * Domain: `Global credentials (unrestricted)`
        * Scope: `Global (Jenkins, nodes, ...)`
        * Kind: `OpenShift Token for OpenShift Client Plugin`
        * Token: `service account token from "oc sa get-token jenkins" command`
        * ID: `jenkins-prod`
        * Description: `Jenkins production service account token`
        * Click **Add**
    * Select the new credentials entry from the drop-down.
    * Disable TLS Verify: `Checked`
    * Server Certificate Authority: `Leave Empty`
    * Default Project: `cicd`
    * Click **Save**

Next, use the same token to create a standare "username and password" credential for the `oc image mirror` command to use in order to connect to the "production" docker image registry.

1. Navigate to **Jenkins -> Credentials -> (global) -> Add Credentials**
    * Kind: `Username with password`
    * Username: `jenkins`
    * Password: `service account token from "oc sa get-token jenkins" command`
    * ID: `image-creds-prod`
    * Click **Save**

You will also need credentials from the "non-prod" cluster in order to copy images.  You will need to create a service account in the non-prod cluster and give it access to the project where your images exist.

If Jenkins already exists in the non-prod cluster, then it would be a good idea to re-use that service account (it should already exist).  If Jenkins and your images both exist in the "cicd" namespace, then simply execute the following commands when logged into the non-prod cluster:

```
oc adm policy add-role-to-user system:image-puller system:serviceaccount:cicd:jenkins -n cicd
```

Then get the service account token:

```
oc sa get-token jenkins -n cicd
```

Create  new credential like you did for the production Jenkins service account:

1. Navigate to **Jenkins -> Credentials -> (global) -> Add Credentials**
    * Kind: `Username with password`
    * Username: `jenkins`
    * Password: `service account token from "oc sa get-token jenkins" command`
    * ID: `image-creds-non-prod`
    * Click **Save**


With this in place, the following pipeline should:

1. List the images in the non-prod cluster "cicd" project.
2. Copy the "petclinic:dev" image/tag from the non-prod "cicd" project to the production "cicd" project.
3. List the image in the production cluster "cicd" project.

This pipeline will run from the non-production cluster.

```
pipeline {
  agent {
    label 'maven'
  }
  stages {
    stage('Initialize') {
      steps {
        echo "Pipeline has started."
      }
    }
    stage("List DEV images") {
      steps {
        script {
          echo "Listing images in non-prod cluster..."
          openshift.withCluster() {
            openshift.withProject('cicd') {
              def istream = openshift.selector('is')
              //istream.describe()
              istream.withEach { // The closure body will be executed once for each selected object.
                    // The 'it' variable will be bound to a Selector which selects a single
                    // object which is the focus of the iteration.
                echo "Image Stream: ${it.name()} is defined in ${openshift.project()}"
              }
            }
          }
        }
      }
    }
    stage('Copy Image') {
    	steps {
    	  withDockerRegistry([credentialsId: "image-creds-non-prod", url: "http://docker-registry.default.svc.cluster.local:5000"]) {
        	withDockerRegistry([credentialsId: "image-creds-prod", url: "https://default-route-openshift-image-registry.rhpds-107036-6c045c923eeb234d1b0c99c65fa6f1d1-0000.us-east.containers.appdomain.cloud"]) {
            sh """
                oc image mirror docker-registry.default.svc.cluster.local:5000/cicd/petclinic:dev default-route-openshift-image-registry.rhpds-107036-6c045c923eeb234d1b0c99c65fa6f1d1-0000.us-east.containers.appdomain.cloud/cicd/petclinic:dev --insecure=true
            """
          }
    	  }
      }
    }
    stage("List PROD images") {
      steps {
        script {
          echo "Listing images in production cluster..."
          openshift.withCluster('production') {
            openshift.withProject('cicd') {
              def istream = openshift.selector('is')
              istream.describe()
              istream.withEach { // The closure body will be executed once for each selected object.
                    // The 'it' variable will be bound to a Selector which selects a single
                    // object which is the focus of the iteration.
                echo "Image Stream: ${it.name()} is defined in ${openshift.project()}"
              }
            }
          }
        }
      }
    }
  }
}
```
