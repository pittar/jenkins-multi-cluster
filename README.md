# Jenkins Multi-Cluster Access

This is a simplified pipeline adapted from [Gerald Nunn's example](https://github.com/gnunn1/openshift-basic-pipeline/tree/master/openshift/cross-cluster).

## Getting Started: Building a Skopeo Jenkins Agent

Out of the box, OpenShift does not have a "Skopeo" Jenkins agent available.  However, it's easy enough to make one from the ["Jenkins Agent Base" image](https://catalog.redhat.com/software/containers/openshift3/jenkins-slave-base-rhel7/581d2f3f00e5d05639b6515b).

In order to create a new "skopeo" Jenkins agent, we only need an "ImageStream" and a "BuildConfig" that can build the image from a Dockerfile.

First, clone this repository, then:

1. Create the Skopeo Jenkins agent `ImageStream` and `BuildConfig` in the same project where Jenkins is installed.  For example, if Jenkins is installed in `cicd-tools`, then run the following command:

```
oc apply -f resources/jenkins-skopeo-agent-imagestream.yaml -n cicd-tools
oc apply -f resources/jenkins-skopeo-agent-buildconfig.yaml -n cicd-tools
```

You can then start the build:

```
oc start-build skopeo -n cicd-tools
```

Once the build completes, you will have a new Jenkins agent named "skopeo".  The `ImageStream` you created includes a label of `role=jenkins-slave`.  This allows Jenkins to auto-discover this new agent type.  To use this agent, you simply need to specify the the agent label of `skopeo` in your Jenkinsfile.

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

Next, use the same token to create a "secret text" credential for the Skopeo command to use in order to connect to the "production" docker image registry.

1. Navigate to **Jenkins -> Credentials -> (global) -> Add Credentials**
    * Kind: `Secret text`
    * Secret: `jenkins:<TOKEN>`
    * ID: `image-creds-prod`
    * Click **Save**

You will also need credentials from the "non-prod" cluster in order to copy images.  You will need to create a service account in the non-prod cluster and give it access to the project where your images exist.

If Jenkins already exists in the non-prod cluster, then it would be a good idea to re-use that service account (it should already exist).  If Jenkins and your images both exist in the "cicd-tools" namespace, then simply execute the following commands when logged into the non-prod cluster:

```
oc adm policy add-role-to-user system:image-puller system:serviceaccount:cicd-tools:jenkins -n cicd-tools
```

Then get the service account token:

```
oc sa get-token jenkins -n cicd-tools
```

Create  new credential like you did for the production Jenkins service account:

1. Navigate to **Jenkins -> Credentials -> (global) -> Add Credentials**
    * Kind: `Secret text`
    * Secret: `jenkins:<TOKEN>`
    * ID: `image-creds-non-prod`
    * Click **Save**


With this in place, the following pipeline should:

1. List the images in the non-prod cluster "cicd-tools" project.
2. Copy the "petclinic:dev" image/tag from the non-prod "cicd-tools" project to the production "cicd" project.
3. List the image in the production cluster "cicd" project.

This pipeline will run from the non-production cluster.

```
pipeline {
  agent none
  stages {
    stage("List DEV images") {
      agent {
        label 'maven'
      }
      steps {
        script {
          echo "Listing images..."
          openshift.withCluster() {
            openshift.withProject('cicd-tools') {
              def istream = openshift.selector('is')
              //istream.describe()
              istream.withEach { // The closure body will be executed once for each selected object.
                    // The 'it' variable will be bound to a Selector which selects a single
                    // object which is the focus of the iteration.
                  echo "Image Stream: ${it.name()} is defined in ${openshift.project()}"
              }
              println "${istream}"
            }
          }
        }
      }
    }
    stage("Skopeo") {
      agent {
        label 'skopeo'
      }
      steps {
        script {
          echo "Skopeo copy step."
          openshift.withCluster() {
            openshift.withProject('cicd-tools') {
              withCredentials([string(credentialsId: 'image-creds-non-prod', variable: 'NON_PROD_CREDS'),
                               string(credentialsId: 'image-creds-prod', variable: 'PROD_CREDS')]) {
            	sh """
            	    set -x
                	skopeo --debug copy \
                	  --src-tls-verify=false \
                	  --dest-tls-verify=false \
                	  --src-creds="${NON_PROD_CREDS}" \
                	  --dest-creds="${PROD_CREDS}" \
                	  docker://docker-registry.default.svc.cluster.local:5000/cicd-tools/petclinic:dev \
                	  docker://<prod cluster registry route>/cicd/petclinic:prod
              	"""
              }
            }
          }
        }
      }
    }
    stage("List PROD images") {
      agent {
        label 'maven'
      }
      steps {
        script {
          echo "Listing images..."
          openshift.withCluster('production') {
            openshift.withProject('cicd') {
              def istream = openshift.selector('is')
              istream.describe()
              istream.withEach { // The closure body will be executed once for each selected object.
                    // The 'it' variable will be bound to a Selector which selects a single
                    // object which is the focus of the iteration.
                  echo "Image Stream: ${it.name()} is defined in ${openshift.project()}"
              }
              println "${istream}"
            }
          }
        }
      }
    }
  }
}
```

When it comes to copying the images, there are a few parts to understand:

* **Local cluster image registry internal URL:** `docker-registry.default.svc.cluster.local:5000`
* **Local image project/imagename:tag:** `/cicd-tools/petclinic:dev`
* **Production cluster image registry exposed route url:** `<prod cluster registry route>`
* **Production image project/imagename:tag:** `/cicd/petclinic:prod`

