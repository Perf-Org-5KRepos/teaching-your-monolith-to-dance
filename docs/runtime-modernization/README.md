# Runtime Modernization Solution

## Table of Contents

- [Introduction](#introduction)
- [Analysis](#analysis) (Reading)
- [Build](#build) (Hands-on)
- [Deploy](#deploy) (Hands-on)
- [Access the Application](#access-the-application) (Hands-on)
- [Summary](#summary)

## Introduction

**Runtime modernization** moves an application to a 'built for the cloud' runtime with the least amount of effort. **Open Liberty** is a fast, dynamic, and easy-to-use Java application server. Ideal for the cloud, Liberty is open sourced, with fast start-up times (<2 seconds), no server restarts to pick up changes, and a simple XML configuration.

Liberty however doesn't support all of the legacy Java EE and WebSphere proprietary functionality and some code changes maybe required to move an existing application to the new runtime. Effort is also required to move the application configuration from traditional WebSphere to Liberty's XML configuration files.

**This path gets the application on to a cloud-ready runtime container which is easy to use and portable. In addition to the necessary library changes, some aspects of the application was modernized. However, it has not been 'modernized' to a newer architecture such as micro-services**.

This repository holds a solution that is the result of a **runtime modernization** for an existing WebSphere Java EE application that was moved from WebSphere ND v8.5.5 to Liberty and deployed by the IBM Cloud Pak for Applications to RedHat OpenShift.

We'll use the same [Customer Order Services application](../common/application.md) from the **Operational Modernization** lab as an example. In order to modernize the runtime, the application will go through **analysis**, **build** and **deploy** phases.

## Analysis

As before, IBM Cloud Transformation Advisor was used to analyze the existing Customer Order Services application and the WebSphere ND runtime. The steps taken were:

1. Used the IBM Cloud Transformation Advisor available as part of IBM Cloud Pak for Applications. Transformation Advisor Local (Beta) can also be used.

2. Downloaded and executed the **Data Collector** against the existing WebSphere ND runtime.

3. Uploaded the results of the data collection to IBM Cloud Transformation Advisor. A screenshot of the analysis is shown below:

    ![Liberty analysis](extras/images/liberty-analyze/analysis1a.jpg)

- In the case of the **CustomerOrderServicesApp.ear** application, IBM Cloud Transformation Advisor has determined that the migration to Liberty on Private Cloud is of **Moderate** complexity and that there are two **Severe Issues** that have been detected.

- Drilling down in to **Detailed Migration Analysis Report** that is part of the application analysis, it is apparent that IBM Cloud Transformation Advisor has detected that there are issues with lookups for Enterprise JavaBeans and with accessing the Apache Wink APIs.

  ![JPA](extras/images/liberty-analyze/severe.jpg)

- **Behaviour change on lookups for Enterprise JavaBeans**: In Liberty, EJB components are not bound to a server root Java Naming and Directory Interface (JNDI) namespace as they are in WebSphere Application Server traditional. The fix for this is to change the three classes that use `ejblocal` to use the correct URL for Liberty

- **The user of system provided Apache Wink APIs requires configuration**: To use system-provided third-party APIs in Liberty applications, you must configure the applications to include the APIs. In WebSphere Application Server traditional, these APIs are available without configuration. This is a configuration only change and can be achieved by using a `classloader` definition in the Liberty server.xml file.

- In summary, some minimal code changes were required to move this application to the Liberty runtime and the decision was taken to proceed with these code changes.

**Homework**: After you complete this workshop, review the step-by-step instructions on how to replicate these steps from the resources included in _Next Steps_ section. Then try Transformation Advisor with one of your applications to migrate it to Liberty.


## Build

In this section, you'll learn how to build a Docker image for Customer Order Services application running on Liberty.

Building this image could take around ~3 minutes (multi-stage build that compiles the code, which takes extra time). As before, let's kick that process off and then come back to learn what you did.

You'll need the web terminal (same as the one from previous lab). If it's not open, follow the instructions [here](../common/setup.md#access-the-web-terminal) to login to OpenShift cluster via the web terminal.

Clone the GitHub repo with the lab artifacts for Liberty (these are different from the artifacts cloned in last lab). Run the following commands on your web terminal:
```
cd / && mkdir liberty && cd liberty
git clone --branch liberty https://github.com/IBM/teaching-your-monolith-to-dance.git
cd teaching-your-monolith-to-dance
ls
```

Run the following command to start building the image. Make sure to copy the entire command, including the `"."` at the end. While the image is building(takes ~3 minutes), continue with rest of the lab:

```
docker build --tag image-registry.openshift-image-registry.svc:5000/apps/cos .
```

### Library changes

- Made the simple code changes required for the EJB lookups which were recommended by IBM Cloud Transformation Advisor. The three Java classes that should be modified to look up Enterprise JavaBeans differently are shown in the detailed analysis view of IBM Cloud Transformation Advisor:

  ![Analysis](extras/images/analysis.jpg)

- Below is an example of the code changes required for one of the three Java classes. The `org.pwte.example.resources.CategoryResource.java` is changed from using `ejblocal` as shown below:

  Before:

    ```java
    ...
    InitialContext().lookup("ejblocal:org.pwte.example.service.ProductSearchService");
    ...
    ```

  After:

  ```java
  ...
  InitialContext().lookup("java:app/CustomerOrderServices/ProductSearchServiceImpl!org.pwte.example.service.ProductSearchService");
  ...
  ```

- Upgraded to Java EE8. Changed from using annotations from _jackson_ to _jsonb_. For example, changed from `@JsonProperty(value="id")` to `@JsonbProperty(value="id")`.


### Modernize with MicroProfile

We used the opportunity to make code changes to modernize some aspects of the application as well. Eclipse MicroProfile is a modular set of technologies designed so that you can write cloud-native microservices. Even though our application is a monolith, we can still take advantage of some of the technologies from MicroProfile.


#### Secure application

We updated the application to use a token-based authentication mechanism to authenticate, authorize, and verify user identities. Added MicroProfile JWT to validate security tokens. The application was updated to use Keycloak, which runs on the cluster and will handle authenticating users. It'll also handle registering & storing user account information.

#### Externalize configuration

The application will have to run on many different environments. So it's important to avoid hardcoding environment specific values in your code. Otherwise, you'll have to update code, recompile and containerize it frequently. 

MicroProfile Config separates the configuration from code. You can inject the external configuration into services in the containers without repackaging them. Applications can use MicroProfile Config as a single API to retrieve configuration information from different sources such as system properties, system environment variables, properties files, XML files, or data sources. Of course, you can do all this by yourself, but it'll be a lot of work and code. Add few MicroProfile Config annotations and you'll make your life easier and code a lot cleaner.

We used MicroProfile Config to [inject information](https://github.com/IBM/teaching-your-monolith-to-dance/blob/6e197f03b8663813ba806b6f321cb9e5ce92c6f6/app/CustomerOrderServicesWeb/src/org/pwte/example/resources/JWTConfigResource.java#L21) about the application's authenticator (Keycloak in this case). For example, added these 3 lines and at runtime the variable will automatically get the value injected by MicroProfile Config:

  ```java
  @Inject
  @ConfigProperty(name = "SSO_URI")
  private String keycloakURI;
  ```

#### Determine application's availability

In the last lab, we used `/CustomerOrderServicesWeb/index.html` for readiness and liveness probes, which is not the best indication that application is ready to handle traffic or is healthy to process requests correctly within a reasonable amount of time. What if the database is down? What if application's security layer is not yet ready/unable to handle authentication? The Pod would still be considered ready and healthy and traffic would still be sent to it. All of those requests will fail or would queue up - leading to bigger problems.

MicroProfile Health provides a common REST endpoint format to determine whether a microservice (or in our case a monolith application) is healthy or not. Health can be determined by the service itself and might be based on the availability of necessary resources (for example, a database) and services. The service itself might be running but considered unhealthy if the things it requires for normal operation are unavailable. All of the checks are performed periodically and the result is served as a simple UP or DOWN at `/health/ready` and `/health/live` which can be used for readiness and liveness probes.

We implemented the following health checks:
- [ReadinessCheck](https://github.com/IBM/teaching-your-monolith-to-dance/blob/6e197f03b8663813ba806b6f321cb9e5ce92c6f6/app/CustomerOrderServicesWeb/src/org/pwte/example/health/ReadinessCheck.java#L17): Keycloak is required to authenticate users. Application should only accept traffic if Keycloak client is up and running.

  ```java
  URL url = new URL(keycloakURI);
  con = (HttpURLConnection) url.openConnection();
  con.setRequestMethod("GET");
  int status = con.getResponseCode();
  if (status != 200) {
    return HealthCheckResponse.named("Readiness").down().build();
  }
  return HealthCheckResponse.named("Readiness").up().build();
  ```

- [LivenessCheck](https://github.com/IBM/teaching-your-monolith-to-dance/blob/f2e9358735a39c5134907e15ae62ba8cb16ad122/app/CustomerOrderServicesWeb/src/org/pwte/example/health/LivenessCheck.java#L15): The requests should be processed within a reasonable amount of time. Monitor thread block times to identify potential deadlocks which can cause the application to hang.

    ```java
    ThreadMXBean tBean = ManagementFactory.getThreadMXBean();
    long ids[] = tBean.findMonitorDeadlockedThreads();
    if (ids !=null) {
      ThreadInfo threadInfos[] = tBean.getThreadInfo(ids);
      for (ThreadInfo ti : threadInfos) {
        double seconds = ti.getBlockedTime() / 1000.0;
        if (seconds > 60) {
          return HealthCheckResponse.named("Liveness").down().build();
        }
      }
    }
    return HealthCheckResponse.named("Liveness").up().build();
    ```

#### Adding metrics to application

MicroProfile Metrics is used to gather metrics about the time it takes to add an item to cart, retrieve customer information and to count the number of time these operations are performed.

  ```java
  @GET
  @Produces(MediaType.APPLICATION_JSON)
  @Counted
  @Timed(name = "getCustomer_timed")
  public Response getCustomer()
  {
  ```

### Liberty server configuration

The Liberty runtime configuration file `server.xml` was created from the template provided by IBM Cloud Transformation Advisor. Have a look at the final version of the file available [here](https://github.com/IBM/teaching-your-monolith-to-dance/tree/liberty/config/server.xml).

  - The necessary features, including those for MicroProfile, are enabled (e.g. `jdbc-4.2, jaxrs-2.1, mpHealth-2.1`).

  - An HTTP endpoint is configured by:
    ```xml
    <httpEndpoint httpPort="-1" httpsPort="9443" accessLoggingRef="accessLogging" id="defaultHttpEndpoint"/>
    ```
  
  - Note that access logging is enabled to record all inbound client requests handled by HTTP endpoint. We'll visualize this data later in dashboard to identify and analyze potential problems. 

  - Application with appropriate security role and classloader visibility is specified by `application` element.

  - Database is configured using the `dataSource` element. Note that Liberty variables are used for certain attributes (e.g. `serverName="${DB_HOST}"`). This will allow the containerized application to be deployed to different environments (e.g. production database vs testing database).

  - The configuration to process the MicroProfile JWT token is defined using `mpJWT` element.

  - `quickStartSecurity` element allows to secure endpoints such as _/metrics_


### Build instructions

The `Dockerfile` required to build the immutable image containing the application and Liberty runtime was created from the template provided by IBM Cloud Transformation Advisor. Here is the final version of the file:

  ```dockerfile
  ## Build stage
  FROM maven:latest AS builder
  COPY app/ /
  RUN cd CustomerOrderServicesProject && mvn clean package

  ## Application image
  FROM openliberty/open-liberty:full-java8-openj9-ubi

  COPY --chown=1001:0 resources/ /opt/ol/wlp/usr/shared/resources/

  COPY --chown=1001:0 config/server.xml /config/
  
  COPY --from=builder --chown=1001:0 CustomerOrderServicesApp/target/CustomerOrderServicesApp-0.1.0-SNAPSHOT.ear /config/apps/CustomerOrderServicesApp.ear

  RUN configure.sh
  ```

  - This is a multi-stage Dockerfile, as indicated by the 2 instructions with `FROM`. The first stage builds the application using Maven. It uses the base image from Maven, copies the application source and then builds using Maven commands. The second stage is the actual application image, which uses the _ear_ file produced by the first stage. 

  - The base image for this application image is `openliberty/open-liberty`, which is the official image for Open Liberty. The tag `full-java8-openj9-ubi` indicates the version of Java and that this image is based on Red Hat's Universal Base Image (UBI). We recommend using UBI images. The `full` keyword indicates that this image comes with additional Liberty features. There is also an image with `kernel`, which comes with the bare minimum server. In this case we are using the latest available image. But you can specify a specific Open Liberty release (for example: `20.0.0.3-full-java8-openj9-ubi`).

  - Copy everything that the application needs into the container including the necessary db2 drivers.
  
  - For security, Liberty containers run as non-root. This is in fact a requirement for running certified containers in OpenShift. The `COPY` instruction by default copies as root. So change user and group using `--chown=1001:0` command.

  - Next, copy Liberty's configuration file `server.xml`.

  - Then copy application ear, produced by the first build stage. This is indicated by the `--from=builder`.

  - As last step run `/configure.sh` which will grow image to be fit-for-purpose.

  Remember that each instruction in the Dockerfile is a layer and each layer is cached. You should always specify the volatile artifacts towards the end.


### Build image

Go back to the web terminal to check on the image build.

You should see the following message if image was built successfully. Please wait if it's still building:

```
Successfully tagged image-registry.openshift-image-registry.svc:5000/apps/cos:latest
```

Validate that image is in the repository by running command:

```
docker images
```

You should see the following images on the output (in addition to the images from previous lab). Notice that the base image, _openliberty/open-liberty_, is also listed. It was pulled as the first step of building application image.

```
REPOSITORY                                                           TAG                     IMAGE ID            CREATED                SIZE
image-registry.openshift-image-registry.svc:5000/apps/cos            latest                  73e50e797849        4 minutes ago          859MB
openliberty/open-liberty                                             full-java8-openj9-ubi   329623a556ff        5 minutes ago          734MB
```

Before we push the image to OpenShift's internal image registry, create a separate project named `apps`. Use the OpenShift console this time to create a project.
- Click on **Home** > **Projects**. 
- Click on `Create Project` button.
- Enter `apps` for the _Name_ field and click on `Create`.

Go back to web terminal. 

Login to the image registry again by running the following command in web terminal:

```
docker login -u openshift -p $(oc whoami -t) image-registry.openshift-image-registry.svc:5000
```

Push the image to OpenShift's internal image registry, which could take up to a minute:

```
docker push image-registry.openshift-image-registry.svc:5000/apps/cos
```

Verify that the image is in image registry:

```
oc get images | grep apps/cos
```

The application image you just pushed should be listed:

```
image-registry.openshift-image-registry.svc:5000/apps/cos@sha256:fbb7162060754261247ad1948dccee0b24b6048b95cd704bf2997eb6f5abfeae
```

In OpenShift console, from the left-panel, click on **Builds** > **Image Streams**. Then select `apps` from the _Project_ drop-down list. Click on `cos` from the list. Scroll down to the bottom to see the image that you pushed.


## Deploy

Let's complete the pre-requisites for the modernized application with Liberty runtime before we deploy it to OpenShift.

Customer Order Services application uses DB2 as its database. To deploy it to Liberty, a separate instance of the database is already configured in the OpenShift cluster you are using. The database is exposed within the cluster using a _Service_ and the application references database using the address of the _Service_.

### Keycloak setup

Keycloak runs on your cluster and will handle authenticating users. It'll also handle registering & storing user account information. Keycloak is the upstream project for Red Hat Single Sign-On (RH-SSO), which is supported as part of IBM Cloud Pak for Applications.

Let's setup a client in Keycloak to handle user authentication for the Customer Order Services application.

1. In OpenShift console, from the left-panel, click on **Networking** > **Routes**.. Then select `keycloak` from the _Project_ drop-down list.

1. Click on the route URL (under the `Location` column) to launch Keycloak.

1. Click on `Administration Console`. Enter `admin` for both username and password.

1. As illustrated in the screen recording below, from the menu options on the left, hover over `Master` and click on `Add realm`. Enter _Galaxy_ for the `Name` field and click on `Create`.
    - A realm manages a set of users, credentials, roles, and groups. A user belongs to and logs into a realm. Realms are isolated from one another and can only manage and authenticate the users that they control.

      ![Keycloak realm](extras/images/keycloak-realm.gif)

1. Configure the realm that you created. In the following two steps, ensure that you click on `Save` before clicking on another tab. 
    1. Click on `Login` tab. Turn on `User registration`. Click on `Save`. This provides new users the option to register.

    1. Click on `Tokens` tab. Set `Access Token Lifespan` to _120 minutes_. Click on `Save`. This specifies the maximum time before an access token is expired.

1. From the menu options on the left, select `Clients`.

1. Click on `Create`. Enter _cos_app_ for `Client ID` field. Click on `Save`.

That concludes the Keycloak setup for now. After we deploy the application, we'll come back to add the route URL of the application as a valid redirect URI. Leave the Keycloak tab open for now.

### Create Secrets

Specifying credentials and tokens in plain text is not secure. `Secrets` are used to store sensitive information. The stored data can be referenced by other resources. Special care is taken by OpenShift when handling data from secrets. For example, they will not be logged or shown anywhere. 

Let's create 2 secrets, one to store database credentials and another for storing Liberty metrics credentials, which is needed to access the `/metrics` endpoint.

In OpenShift console, you can click on the `+` icon on the top panel to quickly create a resource (as shown below). 

  ![create secret](extras/images/create-secret.gif)


Create a `Secret` for database. Click on the `+` icon and paste the following content and click on `Create`.
```yaml
kind: Secret
apiVersion: v1
metadata:
  name: db-creds
  namespace: apps
data:
  DB_USER: ZGIyaW5zdDE=
  DB_PASSWORD: ZGIyaW5zdDE=
type: Opaque
``` 

Let's create the secret for Liberty using the OpenShift command-line interface (CLI) `oc`. So you are familiar with both UI and CLI.

In web terminal, run the following command:

  ```
  oc apply -f https://raw.githubusercontent.com/IBM/teaching-your-monolith-to-dance/liberty/deploy/secret-liberty-creds.yaml
  ```

The `-f` option can specify a file, directory or a URL (as in this case) to use to create the resource. This is the content of the file referenced above:

```yaml
kind: Secret
apiVersion: v1
metadata:
  name: liberty-creds
  namespace: apps
stringData:
  username: admin
  password: admin
type: Opaque
```

Note that the first `Secret` provides the credentials in base64 encoded format using the `data` field. The second one provides in plain text using `stringData` field. OpenShift will automatically convert the credentials to base64 format and place the information under `data` field. Click on the `YAML` tab of `liberty-creds` secret. The `data` field should contain the credentials in encoded form.

As anyone can decode the credentials, administrators should ensure that only authenticated users have access to `Secrets` using Role-based access control (RBAC).

You've completed the pre-requisite steps for deploying the application. 

### Open Liberty Operator

We'll use Open Liberty Operator, available as part of IBM Cloud Pak for Applications, to deploy the application. Open Liberty Operator provides all functionalities of Appsody Operator in addition to Open Liberty runtime specific capabilities, such as day-2 operations (gather trace & dumps) and single sign-on (SSO).

Use the following `OpenLibertyApplication` custom resource (CR), to deploy the Customer Order Services application.

```yaml
apiVersion: openliberty.io/v1beta1
kind: OpenLibertyApplication
metadata:
  name: cos
  namespace: apps
spec:
  applicationImage: 'image-registry.openshift-image-registry.svc:5000/apps/cos'
  pullPolicy: Always
  readinessProbe:
    httpGet:
      path: /health/ready
      port: 9443
      scheme: HTTPS
  livenessProbe:
    httpGet:
      path: /health/live
      port: 9443
      scheme: HTTPS
  service:
    annotations:
      service.beta.openshift.io/serving-cert-secret-name: cos-tls
    certificateSecretRef: cos-tls
    port: 9443
  expose: true
  route:
    termination: reencrypt
    insecureEdgeTerminationPolicy: Redirect
  env:
    - name: SSO_REALM
      value: Galaxy
    - name: SSO_CLIENT_ID
      value: cos_app
    - name: SSO_URI
      value: >-
        https://keycloak-keycloak.ENTER_YOUR_ROUTER_HOSTNAME_HERE/auth/
    - name: JWT_ISSUER
      value: >-
        https://keycloak-keycloak.ENTER_YOUR_ROUTER_HOSTNAME_HERE/auth/realms/Galaxy
    - name: JWT_JWKS_URI
      value: >-
        https://keycloak-keycloak.ENTER_YOUR_ROUTER_HOSTNAME_HERE/auth/realms/Galaxy/protocol/openid-connect/certs
    - name: SEC_TLS_TRUSTDEFAULTCERTS
      value: 'true'
    - name: DB_HOST
      value: cos-db-liberty.db.svc
  envFrom:
  - secretRef:
      name: db-creds
  monitoring:
    endpoints:
      - basicAuth:
          password:
            key: password
            name: liberty-creds
          username:
            key: username
            name: liberty-creds
        interval: 5s
        scheme: HTTPS
        tlsConfig:
          insecureSkipVerify: true
    labels:
      app-monitoring: 'true'
```

- Notice that the parameters are similar to the `AppsodyApplication` custom resource (CR) you used in the operational modernization lab. `OpenLibertyApplication` in addition allows Liberty specific configurations (day-2 operations, single sign-on).

- The application image you pushed earlier to internal image registry is specified for `applicationImage` parameter.

- MicroProfile Health endpoints `/health/ready` and `/health/live` are used for readiness and liveness probes.

- Secured service and route are configured with necessary certificates.

- Environment variables have been defined to be passed on to the running container. Information for the Keycloak client you setup previously is specified using environment variables (e.g. `SSO_REALM`). Before deploying, you'll replace `ENTER_YOUR_ROUTER_HOSTNAME_HERE` (within the URLs) with the hostname of the router in your cluster.

- The host of the database is specified using its _Service_ address `cos-db-liberty.db.svc` and its credentials are passed in using the _Secret_ `db-creds` you created earlier. The `envFrom` parameter is used to define all of the Secret’s data as environment variables. The key from the _Secret_ becomes the environment variable name.

- Enabled application monitoring so that Prometheus can scrape the information provided by MicroProfile Metric's `/metrics` endpoint in Liberty. The `/metrics` endpoint is protected, hence the credentials are provided using the _Secret_ `liberty-creds` you created earlier.


### Deploy application

1. Before deploying the application, enable monitoring by adding the necessary label to namespace as shown below in the screen recording. In OpenShift console, from the left-panel, click on **Administration** > **Namespaces**. Click on the menu-options for `apps` namespace and click on `Edit Labels`. Copy and paste `app-monitoring=true` into the text box and click `Save`.

    ![Add label to namespace](extras/images/add-monitor-label.gif)

1. From the left-panel, click on **Operators** > **Installed Operators**.

1. Ensure that `apps` is selected from the `Project` drop-down list. 

1. You should see `Open Liberty Operator` on the list. From the `Provided APIs` column, click on `Open Liberty Application`.

1. Click on `Create OpenLibertyApplication` button.

1. Delete the default template. Copy and paste the above `OpenLibertyApplication` custom resource (CR). Under `env` variables, replace the 3 instances of `ENTER_YOUR_ROUTER_HOSTNAME_HERE` (within the URLs) with the hostname of the router in your cluster. To get just the hostname, run the following command in web terminal and then copy the output value:

    ```
    oc get route -n keycloak -o yaml | grep Hostname | sed "s/^.*: //"
    ```

    - The hostname is stored inside the route. Above command finds the line with hostname and retrieves its value.
    - Example hostname value: _leojc-wug-cluster-c53a941250098acc3d804eba23ee3789-0000.tor01.containers.appdomain.cloud_ 

1. Click on `Create` button.

1. Click on `cos` from the list. 

1. Navigate down to `Conditions` section and wait for `Reconciled` type to display `True` in Status column. This means Open Liberty Operator had processed the configurations you specified.

1. Click on the `Resources` tab. The resources that the operator created will be listed: _Deployment_, _Service_ and _Route_.

1. On the row with `Deployment` as `Kind`, click on `cos` to get to the Deployment.

1. Click on `Pods` tab. 

1. Wait until the `Status` column displays _Running_ and `Readiness` column displays _Ready_. These indicate that the application within the container is running and is ready to handle traffic.

1. In OpenShift console, from the left-panel, select **Networking** > **Routes**. Ensure that `apps` is selected from the _Project_ drop-down list. You should see the _Route_ named `cos`. As shown in the screen recording below, click on the menu option on the right-side and click on `Delete Route`. On the prompt, click on `Delete` to confirm. You will see that the Route is deleted. But within a split second, another instance of the Route is created. 

    ![delete route](extras/images/delete-route.gif)

    Open Liberty Operator monitors the resources it creates and takes necessary actions if they are not in the desired state you defined in `OpenLibertyApplication` CR. In this case, you requested to expose your application outside the cluster by setting `expose: true` and the operator created the Route resource to achieve that. When it detected that the Route resource was deleted, it instantly created another one. Open Liberty Operator acts as a guardian angel, always watching over your application. Awesome, isn't it?

### Complete Keycloak setup

1. In OpenShift console, from the left-panel, select **Networking** > **Routes**.

1. Ensure that `apps` is selected from the _Project_ drop-down list. Right-click on the Route URL and copy link address. 

1. Go back to Keycloak console. If you had closed the Keycloak tab then select `keycloak` from the _Project_ drop-down list and click on the route URL. If prompted, enter `admin` for both username and password.

1. From the left-panel, click on `Clients`. Click on _cos_app_ under `Client ID` column.

1. Paste the route URL into `Valid Redirect URIs` field and add `*` at the end of the value. Ensure that the value ends with `/*` (as shown below).

1. Enter `+` into `Web Origins` field. This is necessary to enable Cross-Origin Resource Sharing (CORS).

    ![create secret](extras/images/keycloak-valid-uri.png)

1. Apply the changes by clicking on the `Save` button at the bottom.


### Access the application

1. Click on the route URL for the application.

1. Add `/CustomerOrderServicesWeb` to the end of the URL.

1. You'll be taken to the login form. Click on `Register` to register as a new user. Enter the information. Remember your username and password.

1. Click on `Register`. Once a user registers with a realm, they can be granted access to different applications that are authenticated by the same realm. This is useful for managing user authentication for multiple applications within your enterprise.

1. Now you'll be taken back to the _Customer Order Service_ application.

1. From the `Shop` tab, add few items to the cart. Click on an item and then drag and drop the item into the shopping cart. Add at least 5 items to the cart.

1. As the items are added, it'll be shown under _Current Shopping Cart_ (on the right side).

[comment]: <> (Optional: Delete a pod to see how quickly another one is created and becomes ready - compared to traditional WAS, it's much faster)


## Summary

Congratulations! You've completed the second section of the workshop! 

This application has been modified from the initial WebSphere ND v8.5.5 version to run on modern & cloud-native runtime Open Liberty and deployed by IBM Cloud Pak for Applications to RedHat OpenShift.

Do you see the application dancing and busting some moves? How about that moonwalk?

Let's continue with the workshop. Head over to the [Application Management lab](../application-management/README.md).
