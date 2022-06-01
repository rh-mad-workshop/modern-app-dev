# Application Modernization and Migration enablement: Konveyor End to End demo application


## Architecture

The demo includes the following services:

- **Customers**: The original Retail application from which the rest of microservices have been carved out. It still retains the business logic related to customer management. This legacy application runs on Tomcat and uses an Oracle database.
- **Inventory**: Stores detailed information about products. Developed using Quarkus and PostgreSQL as data store. This service has been configured with the the JDK build mode for Quarkus by default.
- **Orders**: Manages all order related entities. It stores only UIDs to refer to Products and Customers. Implemented with Spring Boot and using a PostgreSQL database.
- **Gateway**: Access and aggregation layer for the whole application. It gets orders data and aggregates Products and Customers detailed information. Also implemented with the Spring Boot/PostgreSQL stack.
- **Frontend**: A new front end layer developed with the React flavor of Patternfly, published on Nginx.

![Architecture Screenshot](docs/images/architecture.png?raw=true "Architecture Diagram")

It can be argued that the domain is too fine grained for the modeled business, or that the approach is not optimal for data aggregation. While these statements might be true, the focus on the demo was to present a simple case with microservices interacting with each other, and shouldn't be considered a design aimed for a production solution.

## Setup

This demo has been developed using the following setup:

- OpenShift Container Platform 4.9
- Red Hat OpenShift Pipelines Operator 1.6.2 (Tekton 0.28.3)
- Red Hat OpenShift GitOps Operator 1.4.3

Other setups may work as well, but Tekton 0.28.x or later is required for the pipeline and tasks to work.

## Manual deployment in OCP

The *customers-tomcat-ocp* directory from this repository contains the Customers Tomcat application with all the required changes for it to run in OCP. Inside the *ocp* subdirectory there is also a manifests file with the definition of the required objects to deploy it manually in OCP. In order to do that, the application image will have to be built locally and pushed to the OCP internal registry. It is important to note that the application WAR artifact has to be built with the following command before building the application:

```
mvn clean package -P kubernetes
```

Using the *kubernetes* profile in the build is essential for the application to pick up the configuration file injected via a secret in the pod. That secret has to be also created manually with the following command from the *customers-tomcat-ocp* directory:

```
oc create secret generic customers-secret --from-file=src/main/resources/persistence.properties
```

Make sure that the configuration in the persistence.properties file points to the correct database when creating the secret.

## Deployment Pipeline in OCP

As stated before, one of the main focus areas of this demo is to showcase a modern approach for CI/CD using a set of tools and practices around the GitOps paradigm. For that, a Deployment Pipeline for the Customers application has been developed using Tekton. The following diagram depicts all tasks to be executed by the pipeline and its interaction with external systems and tools:


![Pipeline Screenshot](docs/images/pipeline.png?raw=true "Pipeline Diagram")

Each of these tasks can be described as follows:

- **Clone Repository** downloads the source code from the target Git repository.

- **Build from Source** builds the application artifact from source code. This task has been tweaked to allow selecting the target subdirectory from the repository in which the target application source is available, allowing to have several application/components available in a single repository. **This way of versioning different services/components is highly discouraged**, as the optimal approach would be to have a dedicated repository for each component, since their lifecycle should be independent. Nevertheless, this choice was made to gather all demo materials on a single repository for simplicity purposes.

- **Build image** uses the Dockerfile packaged on the root directory of an application to build and image and push it to the target registry. The image will be tagged with the short commit hash of the source it contains.

- **Update Manifest** uses the short commit hash tag to update the application manifests in Git and point to the newly built image. Application deployment is then delegated to ArgoCD, which is continuously polling the configuration repository for changes and creates/updates all OpenShift objects accordingly.

The pipeline accepts the following parameters:

- **git-url**: URL of the target Git repository.
- **git-branch**: target branch to work with. Defaults to main.
- **app-subdir**: Subdirectory from the repository in which the application source code is stored.
- **target-namespace**: Namespace/project in which to deploy the application.
- **target-registry**: Registry to push the built image to. Defaults to the internal OCP registry (image-registry.openshift-image-registry.svc:5000).


### Pushing to a Git repository with credentials

As the pipeline requires pushing to a Git repository to update manifests and trigger Argo CD synchronizations, it is necessary to provide credentials for the git related tasks (in this case the update-manifest task). For this, a secret has to be created with valid credentials to push to the application repository. This secret [requires a concrete format for Tekton to create the git credentials in the containers properly](https://docs.openshift.com/container-platform/4.9/cicd/pipelines/authenticating-pipelines-using-git-secret.html#op-configuring-basic-authentication-for-git_authenticating-pipelines-using-git-secret), and a YAML file with its definition has been included as well in the *tekton* directory of this repository, looking as follows:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-user-pass
  annotations:
    tekton.dev/git-0: <REPOSITORY DOMAIN, FOR EXAMPLE https://github.com>
type: kubernetes.io/basic-auth
stringData:
  username: <USERNAME WITH PERMISSION TO PUSH TO THE TARGET REPOSITORY>
  password: <PASSWORD>
```

Edit the file and enter valid credentials for your repository and its domain (for example https://github.com if it is hosted in GitHub).

### Accessing registry.redhat.io

The Customers application uses the [JBoss Web Server 5.6 (OpenJDK11) on UBI 8](https://catalog.redhat.com/software/containers/jboss-webserver-5/jws56-openjdk11-openshift-rhel8/610be90f6bbb00c64eecdaf3) image available in registry.redhat.io. As [this registry requires authentication](https://access.redhat.com/RegistryAuthentication), a service account linked to a pull secret for the registry is required to pull images from it in the buildah task [as stated in the Tekton documentation](https://tekton.dev/docs/pipelines/auth/#configuring-docker-authentication-for-docker). In order for it to work, first create a [registry service account for registry.redhat.io using a Red Hat account](https://access.redhat.com/terms-based-registry/) and download the associated pull secret, which should look similar to this:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-red-hat-account-pull-secret
data:
  .dockerconfigjson:
  <contents of the generated config.json file in base64>
type: kubernetes.io/dockerconfigjson
```

Once the file is downloaded, create the secret in OCP with the following command:

```
oc create -f my-red-hat-account-pull-secret.yaml
```

### Associating the git and pull secrets to the pipeline service Account

Once the secrets to access the Git repository and the Red Hat registry have been created, the next step to allow the pipeline to access them is associating the secrets to the *pipeline* service account that is used by default in OCP to execute pipelines. It can be done with the following command:

```
oc patch serviceaccount pipeline -p '{"secrets": [{"name": "my-red-hat-account-pull-secret"},{"name": "git-user-pass"}]}'
```

### Creating pipeline resources

Once the secrets have been setup following the instructions of the previous points, all pipeline resources can be created easily with a single command. From the root directory of this repository, execute the following command:

```
oc create -f ./customers-tomcat-gitops/tekton
```

### Running the Pipeline

Once all setup is done, the pipeline will have to be run to build the image for the Customer application, indicating the customers-tomcat-gitops component directory. In order to run the pipeline, [the tkn CLI will have to be installed](https://docs.openshift.com/container-platform/4.9/cli_reference/tkn_cli/installing-tkn.html). For example, the command and interaction to provide the parameters would be the following:

```
$ tkn pipeline start customers-deployment-pipeline
? Value for param `git-url` of type `string`? https://github.com/rromannissen/appmod-enablement.git
? Value for param `git-branch` of type `string`? (Default is `main`) main
? Value for param `app-subdir` of type `string`? customers-tomcat-gitops
? Value for param `target-namespace` of type `string`? (Default is `retail`) retail
? Value for param `target-registry` of type `string`? (Default is `image-registry.openshift-image-registry.svc:5000`) image-registry.openshift-image-registry.svc:5000
Please give specifications for the workspace: ws
? Name for the workspace : ws
? Value of the Sub Path :  
?  Type of the Workspace : pvc
? Value of Claim Name : customers-pvc
```

A pvc for the customers service has been provided in the Tekton configuration to avoid having to download dependencies on each run of the pipeline. The definition for this pvc is also available in the *customers-tomcat-gitops/tekton* subdirectory in this repository.

## Application Configuration Model for GitOps

Both application and deployment configuration for each component have been modeled using Helm Charts. Each component directory contains a **helm** subdirectory in which the chart is available. All deployment configuration is included in the **values.yaml** available in the root of the **helm** directory. Application configuration is available as configuration files in both the **config** and **secret** subdirectories within the Helm Chart. Non sensitive parameters should be included in the files inside the **config** subdirectory. Sensitive data such as passwords should be included in the files available in the **secret** subdirectory.

This chart will create a ConfigMap containing the files inside the **config** subdirectory to be consumed by the applications via the Kubernetes API. Along with that, a Secret containing the files inside the **secret** subdirectory will be created as well, and mounted in the pod as a volume for the component to access the configuration files directly.


### Granting permission for applications to access the Kubernetes API

Most application components require access to the Kubernetes API in order to access the ConfigMap objects to obtain their configuration at startup. To do this, simply add the view role to the default service account:

```
kubectl create rolebinding default-view --clusterrole=view --serviceaccount=<NAMESPACE>:default --namespace=<NAMESPACE>
```

### Application databases

As stated before, Orders and Inventory services require a PostgreSQL database instance, so the PosgreSQL Ephemeral template can be used. Data initialization is performed at application startup from import.sql and load.sql files.

> **Note**: Object management for the required databases has been kept outside the application Helm charts for simplicity purposes. They will have to be created manually using the available templates in OCP prior to the application deployment or some components will fail to start properly or won't start at all.

## Running Locally

Applications can also run locally for testing purposes. In this case, the command to be used varies between Spring Boot and Quarkus. In the first case, the command is as follows:

```
mvn clean spring-boot:run -P local
```

For Quarkus services, the command is the following:

```
./mvnw compile quarkus:dev -P local
```

The Tomcat application will require a local Tomcat instance.

## Known issues

### SQLFeatureNotSupportedException exception at startup

The following exception is displayed at startup for the Orders service:

```
java.sql.SQLFeatureNotSupportedException: Method org.postgresql.jdbc.PgConnection.createClob() is not yet implemented.
```

This is caused by [an issue in Hibernate that has been fixed in version 5.4.x](https://hibernate.atlassian.net/browse/HHH-12368). Since the Hibernate version used for the Orders service is 5.3.14, a warning is displayed including the full stack trace for this exception. Although annoying, this warning is harmless for this example and can be ignored.
