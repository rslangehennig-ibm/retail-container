---

copyright:
years: 2020,2022
lastupdated: "2022-08-12"

---
<!-- Do not translate or change the format of the text above -->

# How to deploy your applications on Red Hat &trade; OpenShift 4.x

**The latest version of this document is available at [IBM Docs](https://www.ibm.com/docs/en/cta?topic=ma-how-deploy-your-app-red-hat-openshift-container-platform)**   

The following describes how to use the Transformation Advisor migration artifacts to deploy your applications to a Liberty container and to an OpenShift cluster.       
&nbsp;

You have the option to:
- [deploy a single application to a Liberty container](#how-to-deploy-a-single-application).
- [deploy multiple applications from a cluster or group to a Liberty container](#how-to-deploy-a-cluster-or-group-of-applications).

# How to deploy a single application

## A binary project
The following section describes how to deploy a binary based project to Red Hat&trade; OpenShift. For the binary project, you will provide binary files for the application and any dependencies.

You can migrate your application in three steps. This document will reference the following variables:

- `<BINARY_FILE_LOCATION>` is the location of the binary files for your application, including any dependencies and shared library files.
- `<APPLICATION_FILE>` is the binary file for your application.
- `<DEPENDENCY_FILE_LOCATION>` is the location of any dependencies and shared library files that your application requires.
- `<APP_CONTEXT_ROOT>` is the context root for your application. If not defined elsewhere, for example in ibm-web-ext-xml, this corresponds to the `name` attribute in the `<application>` element of the server.xml.
- `<APPLICATION_NAME>` is the name of the application.
- `<CONTAINER_ID>` is the container ID for your docker image. To get this value, enter: docker ps
- `<IMAGE_REFERENCE>` is the reference for this image, including the registry, repository and tag, e.g. docker.io/myspace/myappimage:1.0.0
- `<LIBERTY_HOME>` is the location where you have installed Liberty.
- `<LIBERTY_MACHINE>` is the machine where you have installed the Liberty profile.
- `<MIGRATION_ARTIFACTS_HOME>` is the location where you have unzipped the Transformation Advisor artifacts, or cloned the repository.
- `<OCP_PROJECT>` is the name of the OpenShift project where you want to install the application.

### STEP ONE: Migrate the Java application to Liberty
In this step you will migrate your application to a local Liberty server. This will allow you to verify that your application works correctly on Liberty and make any configuration changes where necessary. After you have tested to make sure your application works on Liberty, you will be ready to create a Liberty container for your application.

### Step 1 Prerequisites
- A Liberty installation    
  &nbsp;
   - You can get WebSphere Liberty here:
     https://www.ibm.com/support/knowledgecenter/SSEQTP_liberty/com.ibm.websphere.wlp.doc/ae/twlp_inst.html     
     **Note**: If your application requires Java EE6 features you will need Websphere Liberty     
     &nbsp;
   - You can get Open Liberty here:    
     https://openliberty.io/    
     &nbsp;
- A copy of the migration artifacts that you downloaded from Transformation Advisor available on the machine where you have installed Liberty.

### Step 1 Tasks
1. Add your application file to the migration bundle, and remove the placeholder file:
```
cp <BINARY_FILE_LOCATION>/<APPLICATION_FILE> <MIGRATION_ARTIFACTS_HOME>/target/
rm <MIGRATION_ARTIFACTS_HOME>/target/*.placeholder
```
2. Add your dependency file(s) to the migration bundle, and remove the placeholder file(s):
```
cp <DEPENDENCY_FILE_LOCATION>/* <MIGRATION_ARTIFACTS_HOME>/src/main/liberty/lib
rm <MIGRATION_ARTIFACTS_HOME>/src/main/liberty/lib/*.placeholder
```
     NOTE: The placeholder files provide the name(s) of all depednecies for your application
3. Create a server in the Liberty installation to run your application:
```
cd <LIBERTY_HOME>/bin
./server create server1
```
4. Go to the location of your migration artifacts and copy the application binary (ear/war) in the target directory to the apps directory of Liberty:
```
cd <MIGRATION_ARTIFACTS_HOME>
cp target/*.war <LIBERTY_HOME>/usr/servers/server1/apps
```
5. If it doesn't already exist, create the directory
```
<LIBERTY_HOME>/usr/shared/config/lib/global
```
and copy any additional binaries from the migration location to that location:
```
mkdir -p <LIBERTY_HOME>/usr/shared/config/lib/global
cp src/main/liberty/lib/* <LIBERTY_HOME>/usr/shared/config/lib/global
```
6. Update the server.xml if necessary:
   - When Transformation Advisor creates the server.xml file for the migration bundle, sensitive data is extracted as variables at the end of the `server.xml` file with an empty default values. Ensure the variable default values are set appropriately.
   - If there are any additional binaries listed in this file that you do not need, remove any reference to them.
 
NOTE: Only set default value for the variables defined in the server.xml file.  If the value is set in the server.xml file,  they cannot be overrode during the deployment time. 

7. Copy the generated server.xml into place, replacing the default server.xml:
```
cp src/main/liberty/config/server.xml <LIBERTY_HOME>/usr/servers/server1/server.xml
```
8. Start the Liberty server:
```
<LIBERTY_HOME>/bin/server start server1
```
9. Check the Liberty logs to confirm that your application has started correctly and to find the URL to for access:
```
cd <LIBERTY_HOME>/usr/servers/server1/logs
vi messages.log
```
NOTE: You may have defined features in the `server.xml` that are not installed by default in Liberty. These will be listed in the log. In this case install the necessary features using `bin/featureUtility`

NOTE: If you define a `<dataSource>` element in the server.xml, you may encounter an authentication issue similar to this:
`invalid username/password; logon denied`

If you see this issue, you may need to update the default value for the auth data variables used in your data source properties. 

10. Verify that the logs contain a line similar to this:
    TCP Channel defaultHttpEndpoint has been started and is now listening for requests on host * (IPv6) port 9080
    If you do not see this then there has been some problem starting the server or launching the application. Search through the log file for more details and debug accordingly.
11. Open your application in the browser by going to the following link:
    `http://<LIBERTY_HOME_MACHINE_IP>:9080/<APP_CONTEXT_ROOT>`

NOTE: The migration artifacts assist you in the migration of your application. Depending on the nature and complexity of your application, additional configuration may be required to fully complete this task. For example, you may need to complete extra configuration to connect your application with a user registry, and or to configure user security role bindings. Consult the product documentation for more details.

### STEP TWO: Containerize Liberty
In this step you will containerize your working Liberty installation. You will create a Liberty image that has your migrated application installed and working, and then test the image to confirm that it is operating correctly.

`NOTE: If you are using podman instead of docker simply replace the word docker in each command with podman`

### Step 2 Prerequisites
- You have completed Step 1: Migrate the Java application to Liberty
- Docker or podman is installed.
   - Download Docker: https://www.docker.com/get-started
   - Download podman: https://podman.io/getting-started/installation
- The machine where you complete this task requires access to the internet to download the Liberty base image.

### Step 2 Tasks
1. Stop the Liberty server if it is running to ensure that the necessary ports are available:
```
<LIBERTY_HOME>/bin/server stop server1
```
2. If using docker ensure the docker service is running. If it’s not, start it:
```
service docker start
```
3. Go to where your migration artifacts are located and build your image from the docker file:
```
cd <MIGRATION_ARTIFACTS_HOME>
docker build --no-cache -t "<IMAGE_REFERENCE>" -f Containerfile .
```
Note: The migration bundle includes a pom.xml file that allows the application and all dependencies to be pulled from Maven.  You need to enable this option in the Containerfile and update the placeholders in the pom.xml file with correct values.
The details of how to do this can be found in the '[Add dependencies using Maven](#add-dependencies-using-maven)' section

4. Run the image and confirm that it is working correctly:
```
docker run –p 9080:9080 <IMAGE_REFERENCE>
```
5. If everything looks good, the image has been started and mapped to the port 9080. You can access it from your browser with this link:
   `http://<LIBERTY_HOME_MACHINE_IP>:9080/<APP_CONTEXT_ROOT>`
   **Optional:** Check your image when it is up and running by logging into the container:
```
docker exec –ti <CONTAINER_ID> bash
```
This allows you to browse the file system of the container where your application is running.

### STEP THREE: Deploy your image to Red Hat OpenShift
In this step you will deploy the image you have created to Red Hat OpenShift. These instructions relate to OpenShift 4+ and have been validated on OpenShift 4.10.

### Step 3 Prerequisites

- Access to either a public or private Red Hat OpenShift 4+ environment.
- Image registry access
   - You will need push your migrated application image to a location that is accessible to the OpenShift cluster. You may use a publicly available registry (e.g. Dockerhub), or your own private registry.
     The migration artifacts generated by Transformation Advisor (specifically the application-cr.yaml file) need to be updated with the image reference that you are using.
     If you do not have a suitable registry available you can create your own in Dockerhub or Podman to use until a suitable registry is found.
     - How to create your own registry in Dockerhub can be found [here](https://www.docker.com/blog/how-to-use-your-own-registry-2/)
     - How to create your own registry in Podman can be found [here](https://www.redhat.com/sysadmin/simple-container-registry)
- A copy of the migration artifacts that you downloaded from Transformation Advisor available on the machine where you have installed Liberty.
- WebSphere Liberty Operator or Open Liberty Operator
   - In order to deploy your application on WebSphere Liberty on OpenShift, you must first have installed the WebSphere Liberty operator on your cluster.
      - The WebSphere Liberty Operator is available in the IBM Operator Catalog. Go to the Operators...OperatorHub UI and click the **IBM Operator Catalog** `Source`. You can use the search to quickly find the WebSphere Liberty operator.
      - Click on the WebSphere Liberty operator UI and follow the instructions to install.
      - You will be given the option of installing the operator cluster wide which will be capable of installing WebSphere Liberty applications across all namespaces, OR, you can choose to install the operator in a given namespace. Choose either option depending on your preference.
      - For full details on the WebSphere Liberty operator, please consult the documentation [here](https://ibm.biz/wlo-docs).
      - For details on how to install the IBM Operator Catalog, please consult the documentation [here](https://github.com/IBM/cloud-pak/blob/master/reference/operator-catalog-enablement.md).
   - In order to get support for your application deployed on Open Liberty on OpenShift, you need to use WebSphere Liberty operator.  But you can still switch to use the Open Liberty operator to do the deployment.
      - you must first have installed the Open Liberty operator on your cluster.
      - The Open Liberty Operator is available in the Certified Catalog. Go to the Operators...OperatorHub UI and click the **Certified** `Source`. You can use the search to quickly find the Open Liberty operator.
      - Click on the Open Liberty operator UI and follow the instructions to install.
      - You will be given the option of installing the operator cluster wide which will be capable of installing Open Liberty applications across all namespaces, OR, you can choose to install the operator in a given namespace. Choose either option depending on your preference.
      - For full details on the Open Liberty operator, please consult the documentation [here](https://github.com/OpenLiberty/open-liberty-operator/tree/main/doc)

### Step 3 Tasks
1. Push your application image to a registry that is accessible to the OpenShift cluster. e.g.
    ```
    docker push <IMAGE_REFERENCE>
    ```
   where `<IMAGE_REFERENCE>` is the image reference you tagged the image with when building, e.g. `docker.io/myspace/myappimage:1.0.0`

   Note:
   - You may need to log in to that image registry in order to successfully push.
   - If your registry requires authenticated pulls, you will need to set up your cluster with a pull secret. See [docs](https://docs.openshift.com/container-platform/4.6/openshift_images/managing_images/using-image-pull-secrets.html) for more information.


2. Log in to the OpenShift cluster and create a new project in OpenShift.

    ```
    oc login -u <USER_NAME>
    ```

   If you have not already created a namespace when installing the Liberty operator create one now.
    ```
    oc new-project <OCP_PROJECT>
    ```

3. Deploy your image with its accompanying operator using the following instructions:

   a. Change to the kustomize directory in `<MIGRATION_ARTIFACTS_HOME>`:

    ```
    cd <MIGRATION_ARTIFACTS_HOME>/deploy/kustomize
    ```

   b. Update the IMAGE_REFERENCE in the `base/application-cr.yaml` file to match the application image reference you pushed in Task 1.
   
   c. Check the value defined in the `base/<APPLICATION_NAME>-configmap.yaml` file are all correct.
   
   d. Update the `overlays/dev/<APPLICATION_NAME>-secret.yaml` file, to add the value for the sensitive date.  The value need to be base64 encoded.  You can invoke command `echo -n 'your-secret-password' | base64` to get the encoded string for your sensitive data.
   
   e. Create the custom resource (CR) for your Liberty application using `apply -k` command to specify a directory containing kustomization.yaml:
    ```
    oc apply -k overlays/dev
    ```

   f. You can view the status of your deployment by running oc get deployments. If you don’t see the status after a few minutes, query the pods and then fetch the Liberty pod logs:
    ```
    oc get pods 
    oc logs <pod>
    ```

   g. You can now access your application by navigating to the `Networking...Routes` UI in OpenShift and selecting the namespace where you deployed.    
   &nbsp;
4. **Optional:** If you wish to delete your application from OpenShift, you can uninstall using the OpenShift UI. Navigate to the `Operators...Installed operators` in OpenShift to Find the Liberty operator. Click on the `All instances` tab to find your application instance and uninstall as required.

## Add dependencies using Maven

### Overview
The migration bundle includes a pom.xml file that allows the application and all dependencies to be pulled from Maven.  You need to enable this option in the Containerfile and update the placeholders in the pom.xml file with correct values.


### Tasks
1. Navigate to the migration bundle
   ```
    cd <MIGRATION_ARTIFACTS_HOME>
    ```
2. Edit the pom.xml file and update the placeholder values with the appropriate values
   1. Set the correct values for the ```<dependency>``` element
   2. Set the correct values for each of the ```<artifactItem>``` elements    
      &nbsp;
3. Edit the Containerfile so that dependencies are pulled in during the image build
   1. Uncomment the line ```RUN mvn -X initialize process-resources verify```


## A source project
### Overview
When migrating an application, you will often need to make changes to the source code to ensure a successful migration to the new target platform.
The exact nature of the changes will vary from application to application.
Transformation Advisor reports on the changes necessary for each individual application and will classify applications that require code changes as either Moderate or Complex.
To pinpoint exactly where these changes need be made in the code you can use the WebSphere Application Migration Toolkit (WAMT) Eclipse plugin.
The tool can also suggest possible fixes. See this link for more details:
https://developer.ibm.com/wasdev/downloads/#asset/tools-WebSphere_Application_Server_Migration_Toolkit    
The following tasks help you build your source code to an image.

### Tasks
1. Navigate to the migration bundle
   ```
    cd <MIGRATION_ARTIFACTS_HOME>
    ```
2. Edit the pom.xml file and update or delete the placeholder values with the appropriate values
   1. Set the correct values for the ```<dependency>``` element
   2. Set the correct values for each of the ```<artifactItem>``` elements    
      &nbsp;
3. Update the pom.xml file to build the application according to your specifications
   1. NOTE: If you already have a pom.xml for your application then you can use that, or merge with the migration bundle pom.xml as appropriate    
      &nbsp;
4. Edit the Containerfile so that source code will be built during the image build
   1. Uncomment the line ```RUN mvn clean package	```


# How to deploy a cluster or group of applications

This section covers how to use the migration bundle for a cluster or group to deploy multiple applications to a single Liberty container.    
You can migrate each application individually first and then merge them into a single deployment, or you can deploy them all at once as a single first step.

If you are migrating a cluster then it is unlikely that you will get a clash of features between your applications. In this case the "All applications at once approach" will be the fastest approach to migrate your cluster.

If you are migrating a group then it is more likely that you may get a feature clash when deploying the applications together, because the applications may not have run together before. It is recommended to take the "All applications at once approach" but if you encounter issues, it is recommended to take the "Application by application" approach to resolve them.

## All applications at once

You can migrate all your applications in three steps. This document will reference the following variables:

- `<APP_CONTEXT_ROOT>` is the context root for your application. If not defined elsewhere, for example in ibm-web-ext-xml, this corresponds to the `name` attribute in the `<application>` element of the server.xml.
- `<APPLICATION_NAME>` is the name of the application.
- `<MIGRATION_ARTIFACTS_HOME>` is the location where you have unzipped the Transformation Advisor artifacts, or cloned the repository.
- `<IMAGE_REFERENCE>` is the reference for this image, including the registry, repository and tag, e.g. docker.io/myspace/myappimage:1.0.0

### STEP ONE: Gather applications and dependency files and update configuration
In this step you will gather your application files and any dependencies and update the configuration files.

### Step 1 Tasks
1. Add your application file(s) to the migration bundle, and remove the placeholder files:
    ```
    cp <BINARY_FILE_LOCATION>/<APPLICATION_FILE> <MIGRATION_ARTIFACTS_HOME>/target/
    rm <MIGRATION_ARTIFACTS_HOME>/target/*.placeholder
    ```
2. Add your dependency file(s) to the migration bundle, and remove the placeholder file(s):
    ```
    cp <DEPENDENCY_FILE_LOCATION>/* <MIGRATION_ARTIFACTS_HOME>/src/main/liberty/lib
    rm <MIGRATION_ARTIFACTS_HOME>/src/main/liberty/lib/*.placeholder
    ```
   ```NOTE: The placeholder files provide the name(s) of all dependencies for your application```        
   &nbsp;    
   ```NOTE: If you are using maven to import your application and dependencies you can skip task 1 & 2 of this step```       
   &nbsp;
4. Update any application server.xml files if necessary:
   - The server.xml file at `<MIGRATION_ARTIFACTS_HOME>/src/main/liberty/config` contains a series of includes, that include the server.xml files for each application
   - The application server.xml files can be found in the following locations: `<MIGRATION_ARTIFACTS_HOME>/apps/<APPLICATION_NAME>/src/main/liberty/config`
   - The name of the server.xml file for each application will be `<APPLICATION_NAME>_server_config.xml`
   - Modify each server.xml files by entering default values for any sensitive data that Transformation Advisor has removed.
   - If there are any additional binaries listed in the server.xml file that you do not need, remove any reference to them.

NOTE: Only set default value for the variables defined in the server.xml file.  If the value is set in the server.xml file,  they cannot be overrode during the deployment time.


### STEP TWO: Containerize all applications on Liberty
In this step you will containerize your working Liberty installation. You will create a Liberty image that has all your migrated applications installed and working, and then test the image to confirm that it is operating correctly.

`NOTE: If you are using podman instead of docker simply replace the word docker in each command with podman`

### Step 2 Prerequisites
- You have completed Step 1: Gather applications and dependency files and update configuration
- Docker or podman is installed.
   - Download Docker: https://www.docker.com/get-started
   - Download podman: https://podman.io/getting-started/installation
- The machine where you complete this task requires access to the internet to download the Liberty base image.

### Step 2 Tasks
1. If using docker ensure the docker service is running. If it’s not, start it:
```
service docker start
```
2. Go to where your migration artifacts are located and build your image from the docker file:
```
cd <MIGRATION_ARTIFACTS_HOME>
docker build --no-cache -t "<IMAGE_REFERENCE>" -f Containerfile .
```
Note: The migration bundle includes a pom.xml file that allows the application and all dependencies to be pulled from Maven.  You need to enable this option in the Containerfile and update the placeholders in the pom.xml file with correct values.
The details of how to do this can be found in the '[Add dependencies using Maven](#add-dependencies-using-maven)' section.

4. Run the image and confirm that it is working correctly:
```
docker run –p 9080:9080 <IMAGE_REFERENCE>
```
5. If everything looks good, the image has been started and mapped to the port 9080.    
   You can access it from your browser with this link:
   `http://<LIBERTY_HOME_MACHINE_IP>:9080/>`    
   &nbsp;

6. Each application will be available at `http://<LIBERTY_HOME_MACHINE_IP>:9080/<APP_CONTEXT_ROOT>`    
   &nbsp;

7. **Optional:** Check your image when it is up and running by logging into the container:
```
docker exec –ti <CONTAINER_ID> bash
```
This allows you to browse the file system of the container where your applications are running.

### STEP THREE: Deploy your image with all applications to Red Hat OpenShift
In this step you will deploy the image you have created to Red Hat OpenShift. These instructions relate to OpenShift 4+ and have been validated on OpenShift 4.10.
Follow the same steps as if your image had a single application - see [here](#deploy-your-image-to-red-hat-openshift)


## A binary project - application by application
In this approach you will configure, containerize and deploy each application individually, and then deploy them together

### Tasks
1. Complete the steps for '[How to deploy a single application](#how-to-deploy-a-single-application)' for each application.
```
cd <MIGRATION_ARTIFACTS_HOME>/apps
Deploy each application indvidually
```
2. Now deploy all applications together following these [steps](#all-applications-at-once)    
   &nbsp;

NOTE: You may have a feature conflict when you deploy applications together. In this case you need to either move the apps that are causing the conflict into another deployment, or update the applications so they no longer require conflicting features.

# Managing keystores during deployment
``` 
When deploying to OpenShift, if your application is configured to use non default keystores then the default route will not work without configuration changes
```
## Configuring keystores in a container
_NOTE: If you plan to deploy to OpenShift you can use the Operator to generate the necessary certificates, see the next section for details._

Transformation Advisor does not automatically migrate keystore information in the migration bundle.    
When running in a container using the default Containerfile produced by Transformation Advisor the Liberty server will output messages indicating that the non-default keystore files can not be found.

For instructions on configuring keystores in Liberty server, see the Liberty [Configuring Security documentation](https://github.com/OpenLiberty/ci.docker/blob/main/SECURITY.md)

## Configuring keystores in an OpenShift deployment
When deploying to OpenShift you can use the Liberty Operator to generate all necessary certificates or use your own certificates.

### Using the Liberty Operator to generate certificates
Complete the following steps to use the Liberty Operator to generate the certificates for your application

1. Modify the `server.xml` file and remove all `<keystore>` attributes.
2. Build and deploy the image as normal

In this case all certification generation and keystore management will be handled by the Operator.    
Further details on this can be found in the [Liberty documentation](https://www.ibm.com/docs/en/was-liberty/core?topic=applications-configuring-tls-certificates)

### Using your own certificate and keystores in OpenShift
You can configure your deployment to use your own certificates and manage the security yourself.    
Further details on this can be found in the Liberty documentation for [specifying certificates](https://www.ibm.com/docs/en/was-liberty/core?topic=applications-configuring-tls-certificates#cfg-t-sec-cert__sec-route-service)