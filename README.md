This is a minimal setup of cBioPortal for UCLA Pathology developed January 2023

Note the cBioPortal docs for [docker deployment](https://docs.cbioportal.org/deployment/docker/) and [SAML authentication](https://docs.cbioportal.org/deployment/authorization-and-authentication/authenticating-users-via-saml/).

Below is the readme with some added documentation for [`cbioportal-docker-compose`](https://github.com/cBioPortal/cbioportal-docker-compose):

---

# Run cBioPortal using Docker Compose
Download necessary files (seed data, example config and example study from
datahub):
```
./init.sh
```

Start docker containers. This can take a few minutes the first time because the
database needs to import some data.
```
docker-compose up
```
If you are developing and want to expose the database for inspection through a program like Sequel Pro, run:
```
docker-compose -f docker-compose.yml -f open-db-ports.yml up
```
In a different terminal import a study
```
docker-compose exec cbioportal metaImport.py -u http://cbioportal:8080 -s study/lgg_ucsf_2014/ -o
```

Restart the cbioportal container after importing:
```
docker-compose restart cbioportal
```

The compose file uses docker volumes which persist data between reboots. To completely remove all data run:

```
docker-compose down -v
```

If you were able to successfully set up a local installation of cBioPortal, please add it here: https://www.cbioportal.org/installations. Thank you!

### Comprehensive Start

#### Step 1 - Run Docker Compose

Download the git repo that has the Docker compose file and go to the root of that folder:

```
git clone https://github.com/cBioPortal/cbioportal-docker-compose.git
cd cbioportal-docker-compose
```

Then download all necessary files (seed data, example config and example study from datahub) with the init script:
```
./init.sh
```

Then run:

```
docker-compose up
```

This will start all four containers (services) defined [here](https://github.com/cBioPortal/cbioportal-docker-compose/blob/master/docker-compose.yml). That is:

- the mysql database, which holds most of the cBioPortal data
- the cBioPortal Java web app, this serves the React frontend as well as the REST API
- the session service Java web app. This service has a REST API and stores session information (e.g. what genes are being queried) and user specific data (e.g. saved cohorts) in a separate mongo database
- the mongo database that persists the data for the session service

It will take a few minutes the first time to import the seed database and perform migrations if necessary. Each container outputs logs to the terminal. For each log you'll see the name of the container that outputs it (e.g. `cbioportal_container` or `cbioportal_session_database_container`). If all is well you won't see any significant errors (maybe some warnings, that's fine to ignore). If all went well you should be able to visit the cBioPortal homepage on http://localhost:8080. You'll notice that no studies are shown on the homepage yet:

<img width="1414" alt="Screen Shot 2022-01-24 at 2 10 10 PM" src="https://user-images.githubusercontent.com/1334004/150848276-dec9551f-6b90-470f-bb59-e7754829fc83.png">


Go to the next step to see how to import studies.

##### Notes on detached mode

If you prefer to run the services in detached mode (i.e. not logging everything to your terminal), you can run

```
docker-compose up -d
```

In this mode, you'll have to check the logs of each container manually using e.g.:

```
docker logs -f cbioportal_container
```

You can list all containers running on your system with

```
docker ps -a
```

#### Step 2 - Import Studies
To import studies you can run:

```
docker-compose run cbioportal metaImport.py -u http://cbioportal:8080 -s study/lgg_ucsf_2014/ -o
```
This will import the [lgg_ucsf_2014 study](https://www.cbioportal.org/patient?studyId=lgg_ucsf_2014) into your local database. It will take a few minutes to import. After importing, restart the cbioportal web container:

```
docker-compose restart cbioportal
```

or 

All public studies can be downloaded from https://www.cbioportal.org/datasets, or https://github.com/cBioPortal/datahub/. You can add any of them to the `./study` folder and import them. There's also a script (`./study/init.sh`) to download multiple studies. You can set `DATAHUB_STUDIES` to any public study id (e.g. `lgg_ucsf_2014`) and run `./init.sh`.

##### Notes on restarting
To avoid having to restart one can alternatively hit an API endpoint. To do so, call the `/api/cache` endpoint with a `DELETE` http-request (see [here](/deployment/customization/portal.properties-Reference.md#evict-caches-with-the-apicache-endpoint) for more information):

```
curl -x DELETE -H "X-API-KEY: my-secret-api-key-value" http://localhost:8080/api/cache
```

The value of the API key is configured in the _portal.properties_ file. You can visit http://localhost:8080 again and you should be able to see the new study.

#### Step 3 - Customize your portal.properties file ###

The properties file can be found in `./config/portal.properties`. Which was set up when running `init.sh`.

This properties file allows you to customize your instance of cBioPortal with e.g. custom logos, or point the cBioPortal container to e.g. use an external mysql database. See the [properties](/deployment/customization/Customizing-your-instance-of-cBioPortal.md) documentation for a comprehensive overview.

If you would like to enable OncoKB see [OncoKB data access](/deployment/integration-with-other-webservices/OncoKB-Data-Access.md) for 
how to obtain a data access token. After obtaining a valid token use:

#### Step 4 - Customize cBioPortal setup
To read more about the various ways to use authentication and parameters for running the cBioPortal web app see the relevant [backend deployment documentation](/deployment/customization/Customizing-your-instance-of-cBioPortal.md).

On server systems that can easily spare 4 GiB or more of memory, set the `-Xms`
and `-Xmx` options to the same number. This should increase performance of
certain memory-intensive web services such as computing the data for the
co-expression tab. If you are using MacOS or Windows, make sure to take a look
at [these notes](notes-for-non-linux.md) to allocate more memory for the
virtual machine in which all Docker processes are running.

## More commands ##
For documentation on how to import a study, see [this tutorial](import_data.md)
For more uses of the cBioPortal image, see [this file](example_commands.md)

To Dockerize a Keycloak authentication service alongside cBioPortal,
see [this file](using-keycloak.md).

## Uninstalling cBioPortal ##
```
docker compose down -v --rmi all
```

## Known issues

### Macbook M1

See ticket and solution: https://github.com/cBioPortal/cbioportal/issues/9829

## Loading other seed databases
### hg38 support
To enable hg38 support. First delete any existing databases and containers:
```
docker-compose down -v
```
Then run
```
init_hg38.sh
```
Followed by:
```
docker-compose up
```
When loading hg38 data make sure to set `reference_genome: hg38` in [meta_study.txt](https://docs.cbioportal.org/5.1-data-loading/data-loading/file-formats#meta-file-4). The example study in `study/` is `hg19` based. 


## Example Commands
### Connect to the database
```
docker-compose exec cbioportal-database \
    sh -c 'mysql -hcbioportal-database -u"$MYSQL_USER" -p"$MYSQL_PASSWORD" "$MYSQL_DATABASE"'
```

## Advanced topics
### Run different cBioPortal version

A different version of cBioPortal can be run using docker compose by declaring the `DOCKER_IMAGE_CBIOPORTAL`
environmental variable. This variable can point a DockerHub image like so:

```
export DOCKER_IMAGE_CBIOPORTAL=cbioportal/cbioportal:3.1.0
docker-compose up
```

which will start the v3.1.0 portal version rather than the newer default version.

### Change the heap size
#### Web app
You can change the heap size in the command section of the cbioportal container

#### Importer
For the importer you can't directly edit the java command used to import a study. Instead add `JAVA_TOOL_OPTIONS` as an environment variable to the cbioportal container and set the desired JVM parameters there (e.g. `JAVA_TOOL_OPTIONS: "-Xms4g -Xmx8g"`).

# SAML Authentication

The cBioPortal includes support for SAML (Security Assertion Markup Language).  This document explains why you might find SAML useful, and how to configure SAML within your own instance of cBioPortal.

Please note that configuring your local instance to support SAML requires many steps.  This includes configuration changes and a small amount of debugging.  If you follow the steps below, you should be up and running relatively quickly, but be forewarned that you may have do a few trial runs to get everything working.

In the documentation below, we also provide details on how to perform SAML authentication via a commercial company:  [OneLogin](https://www.onelogin.com/).  OneLogin provides a free tier for testing out SAML authentication, and is one of the easier options to get a complete SAML workflow set-up.  Once you have OneLogin working, you should then have enough information to transition to your final authentication service.

## What is SAML?

SAML is an open standard that enables one to more easily add an authentication service on top of any existing web application.  For the full definition, see the [SAML Wikipedia entry](https://en.wikipedia.org/wiki/Security_Assertion_Markup_Language).

In its simplest terms, SAML boils down to four terms:

* **identity provider**:  this is a web-based service that stores user names and passwords, and provides a login form for users to authenticate.  Ideally, it also provides easy methods to add / edit / delete users, and also provides methods for users to reset their password.  In the documentation below, OneLogin.com serves as the identity provider.

* **service provider**:  any web site or web application that provides a service, but should only be available to authenticated and authorized users.  In the documentation below, the cBioPortal is the service provider.

* **authentication**:  a means of verifying that a user is who they purport to be.  Authentication is performed by the identify provider, by extracting the user name and password provided in a login form, and matching this with information stored in a database. When authentication is enabled, multiple cancer studies can be stored within a single instance of cBioPortal while providing fine-grained control over which users can access which studies.  Authorization is implemented within the core cBioPortal code, and *not* the identify provider.

## Why is SAML Relevant to cBioPortal?

The cBioPortal code has no means of storing user name and passwords and no means of directly authenticating users.  If you want to restrict access to your instance of cBioPortal, you therefore have to consider an external authentication service.  SAML is one means of doing so, and your larger institution may already provide SAML support.  For example, at Sloan Kettering and Dana-Farber, users of the internal cBioPortal instances login with their regular credentials via SAML.  This greatly simplifies user management.

# Setting up an Identity Provider

As noted above, we provide details on how to perform SAML authentication via a commercial company:  [OneLogin](https://www.onelogin.com/).  
If you already have an IDP set up, you can skip this part and go to [Configuring SAML within cBioPortal](#configuring-saml-within-cbioportal).

OneLogin provides a free tier for testing out SAML authentication, and is one of the easier options to get a complete SAML workflow set-up.  Once you have OneLogin working, you should then have enough information to transition to your final authentication service.  As you follow the steps below, the following link may be helpful: [How to Use the OneLogin SAML Test Connector](https://support.onelogin.com/hc/en-us/articles/202673944-How-to-Use-the-OneLogin-SAML-Test-Connector).

To get started:

* [Register a new OneLogin.com Account](https://www.onelogin.com/signup?ref=floating-sidebar)

## Setting up a SAML Test Connector

* [Login to OneLogin.com](https://app.onelogin.com/login).
* Under Apps, Select Add Apps.
* Search for SAML.
* Select the option labeled: OneLogin SAML Test (IdP w/attr).
* "SAVE" the app, then select the Configuration Tab.

* Under the Configuration Tab for OneLogin SAML Test (IdP w/attr), paste the following fields (this is assuming you are testing everything via localhost).

    * Audience: cbioportal
    * Recipient: http://localhost:8080/saml/SSO
    * ACS (Consumer) URL Validator*:  ^http:\/\/localhost\:8080\/cbioportal\/saml\/SSO$
    * ACS (Consumer) URL*:  http://localhost:8080/saml/SSO

![](/images/previews/onelogin-config.png)


* Add at least the parameters: 
    * Email (Attribute)
    * Email (SAML NameID)

![](/images/previews/onelogin-config-parameters.png)

* Find your user in the "Users" menu

![](/images/previews/onelogin-users-search.png)

* Link the SAML app to your user (click "New app" on the **+** icon found on the top right of the "Applications" table to do this - see screenshot below): 

![](/images/previews/onelogin-add-app.png)

* Configure these **email** parameters in the Users menu:

![](/images/previews/onelogin-users.png)
 

## Downloading the SAML Test Connector Meta Data

* Go to the SSO Tab within OneLogin SAML Test (IdP), find the field labeled:  Issuer URL.  Copy this URL and download it's contents.  This is an XML file that describes the identity provider.

![](https://s3.amazonaws.com/f.cl.ly/items/1R0v0L3P1U2E23202V1d/Image%202015-05-24%20at%2010.07.56%20PM.png)

then, move this XML file to:

    portal/src/main/resources/

You should now be all set with OneLogin.com.  Next, you need to configure your instance of cBioPortal.

# Configuring SAML within cBioPortal

## Creating a KeyStore

In order to use SAML, you must create a [Java Keystore](https://docs.oracle.com/javase/7/docs/api/java/security/KeyStore.html).  

This can be done via the Java `keytool` command, which is bundled with Java.

Type the following:

    keytool -genkey -alias secure-key -keyalg RSA -keystore samlKeystore.jks

This will create a Java keystore for a key called:  `secure-key` and place the keystore in a file named `samlKeystore.jks`.  You will be prompted for:

* keystore password (required, for example:  apollo1)
* your name, organization and location (optional)
* key password for `secure-key` (required, for example apollo2)

When you are done, copy `samlKeystore.jsk` to the correct location:
    
    mv samlKeystore.jks portal/src/main/resources/

If you need to export the public certificate associated within your keystore, run:

    keytool -export -keystore samlKeystore.jks -alias secure-key -file cBioPortal.cer

##### HTTPS and Tomcat

:warning: If you already have an official (non-self-signed) SSL certificate, and need to get your site 
running on HTTPS directly from Tomcat, then you need to import your certificate into the keystore instead. 
See [this Tomcat documentation page](https://tomcat.apache.org/tomcat-8.0-doc/ssl-howto.html) for more details.

:warning: An extra warning for when configuring HTTPS for Tomcat: use the same password for 
both keystore and secure-key. This seems to be an extra restriction by Tomcat.


## Modifying configuration

Within portal.properties, make sure that:

    app.name=cbioportal

Then, modify the section labeled `authentication`. See SAML parameters shown in example below:

    saml.sp.metadata.entityid=cbioportal
    saml.sp.metadata.wantassertionsigned=true
    saml.idp.metadata.location=classpath:/onelogin_metadata_620035.xml
    saml.idp.metadata.entityid=https://app.onelogin.com/saml/metadata/620035
    saml.keystore.location=classpath:/samlKeystore.jks
    saml.keystore.password=apollo1
    saml.keystore.private-key.key=secure-key
    saml.keystore.private-key.password=apollo2
    saml.keystore.default-key=secure-key
    saml.idp.comm.binding.settings=defaultBinding
    saml.idp.comm.binding.type=
    saml.idp.metadata.attribute.email=User.email
    saml.custom.userservice.class=org.cbioportal.security.spring.authentication.saml.SAMLUserDetailsServiceImpl
    # global logout (as opposed to local logout):
    saml.logout.local=false
    saml.logout.url=/

Please note that you will have to modify all the above to match your own settings. `saml.idp.comm.binding.type` can be left empty if `saml.idp.comm.binding.settings=defaultBinding`. The `saml.logout.*` settings above reflect the settings of an IDP that supports Single Logout (hopefully the default in most cases - more details in section below).

In the case that you are running cBioPortal behind a reverse proxy that handles the SSL certificates (such as nginx or traefik), you will have to also specify `saml.sp.metadata.entitybaseurl`. This should point to `https://host.example.come:443`. This setting is required such that cBioPortal uses the Spring SAML library appropriately for creating redirects back into cBioPortal.

In addition there is a known bug where redirect from the cBioPortal instance always goes over http instead of https (https://github.com/cBioPortal/cbioportal/issues/6342). To get around this issue you can pass the full URL including https to the `webapp-runnner.jar` command with e.g. `--proxy-base-url https://mycbioportalinstance.org`.

### Custom scenarios

:information_source: Some settings may need to be adjusted to non-default values, depending on your IDP. For example, if your 
IDP required HTTP-GET requests instead of HTTP-POST, you need to set these properties as such:
 
    saml.idp.comm.binding.settings=specificBinding
    saml.idp.comm.binding.type=bindings:HTTP-Redirect

If you need a very different parsing of the SAML tokens than what is done at `org.cbioportal.security.spring.authentication.saml.SAMLUserDetailsServiceImpl`, you can point the `saml.custom.userservice.class` to your own implementation: 

    saml.custom.userservice.class=<your_package.your_class_name>

:warning: The property `saml.idp.metadata.attribute.email` can also vary per IDP. It is important to set this correctly since this is a required field by the cBioPortal SAML parser (that is, if `org.cbioportal.security.spring.authentication.saml.SAMLUserDetailsServiceImpl` is chosen for property `saml.custom.userservice.class`). 

:warning: Some IDPs like to provide their own logout page (e.g. when they don't support the custom SAML Single Logout protocol). For this you can adjust the  
`saml.logout.url` property to a custom URL provided by the IDP. Also set the `saml.logout.local=true` property in this case to indicate that global logout (or Single Logout) is not supported by IDP:

    # local logout followed by a redirect to a global logout page:
    saml.logout.local=true
    saml.logout.url=<idp specific logout URL, e.g. https://idp.logoutpage.com >
    

:warning: Some IDPs (e.g. Azure Active Directory) cache user data for more than 2 hours causing cbioportal to complain that the authentication statement is too old to be used. You can fix this problem by setting `forceAuthN` to true. Below is an example how you can do this with the properties. You can choose any binding type you like. `bindings:HTTP-Redirect` is given just as an example.

```
    saml.idp.comm.binding.settings=specificBinding
    saml.idp.comm.binding.type=bindings:HTTP-Redirect
    saml.idp.comm.binding.force-auth-n=true
```

## More customizations

If your IDP does not have the flexibility of sending the specific credential fields expected by our 
default "user details parsers" implementation (i.e. `security/security-spring/src/main/java/org/cbioportal/security/spring/authentication/saml/SAMLUserDetailsServiceImpl.java` 
expects field `mail` to be present in the SAML credential), then please let us know via a [new 
issue at our issue tracking system](https://github.com/cBioPortal/cbioportal/issues/new), so we can 
evaluate whether this is a scenario we would like to support in the default code. You can also consider 
adding your own version of the `SAMLUserDetailsService` class. 

## Authorizing Users

Next, please read the Wiki page on [User Authorization](User-Authorization.md), and add user rights for a single user.


## Configuring the Login.jsp Page (not applicable to most external IDPs)

The login page is configurable via the `portal.properties` properties `skin.authorization_message` and `skin.login.saml.registration_htm`. 
For example in `skin.authorization_message` you can be set to something like this:

```
skin.authorization_message= Welcome to this portal. Access to this portal is available to authorized test users at YOUR ORG.  [<a href="https://thehyve.nl/">Request Access</a>].
```

and `skin.login.saml.registration_htm` can be set to:

```
skin.login.saml.registration_htm=Sign in via XXXX
```

You can also set a standard text in `skin.login.contact_html` that will appear in case of problems: 

```
skin.login.contact_html=If you think you have received this message in error, please contact us at <a style="color:#FF0000" href="mailto:cbioportal-access@your.org">cbioportal-access@your.org</a>
```


## Doing a Test Run

You are now ready to go.

Rebuild the WAR file and follow the [Deployment with authentication
steps](/deployment/deploy-without-docker/Deploying.md#required-login) using `authenticate=saml`.

Then, go to:  [http://localhost:8080/](http://localhost:8080/).

If all goes well, the following should happen:

* You will be redirected to the OneLogin Login Page.
* After authenticating, you will be redirected back to your local instance of cBioPortal.

If this does not happen, see the Troubleshooting Tips  below.

## Troubleshooting Tips 

### Logging

Getting this to work requires many steps, and can be a bit tricky.  If you get stuck or get an obscure error message,
your best bet is to turn on all DEBUG logging.  This can be done via `src/main/resources/logback.xml`.  For example:

```
<root level="debug">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="FILE" />
</root>

<logger name="org.mskcc" level="debug">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="FILE" />
</logger>

<logger name="org.cbioportal.security" level="debug">
    <appender-ref ref="STDOUT" />
    <appender-ref ref="FILE" />
</logger>
```

Then, rebuild the WAR, redeploy, and try to authenticate again.  Your log file will then include hundreds of SAML-specific messages, even the full XML of each SAML message, and this should help you debug the error.

### Seeing the SAML messages

Another tool we can use to troubleshoot is SAML tracer (https://addons.mozilla.org/en-US/firefox/addon/saml-tracer/ ). You can add this to Firefox and it will give you an extra menu item in "Tools". Go through the loging steps and you will see the SAML messages that are sent by the IDP. 

### Obtaining the Service Provider Meta Data File

By default, the portal will automatically generate a Service Provider (SP) Meta Data File.  You may need to provide this file to your Identity Provider (IP).

You can access the Service Provider Meta Data File via a URL such as:

[http://localhost:8080/saml/metadata](http://localhost:8080/saml/metadata)
