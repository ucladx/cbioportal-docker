# Deploying cBioPortal with SAML Authentication
**TODO**
    - Okta settings
    - Update portal.props settings?
## Overview
This is a guide made by documenting steps to setting up and configuring the [cBioPortal](https://www.cbioportal.org/) tool for secure usage by UCLA Pathology. We also use KeyCloak integrated with our enterprise identity provider Okta to manage users and their level of access to cBioPortal studies.

This guide closely mirrors the [cBioPortal deployment docs](https://docs.cbioportal.org/deployment/), but includes tweaks and small changes specific to this deployment.
### Dependencies
Docker, Java (JRE), Keycloak

## Server Setup
### VM and OS
The technical work here is handled by ISS, but we need to provide them some information:
- Our initial filesystem setup (January 2023). We allocate 60G to `/opt` as we will have Docker store its data there, which can include large images.
```bash
Filesystem                Size  Used Avail Use% Mounted on
devtmpfs                  4.0M  8.0K  4.0M   1% /dev
tmpfs                     7.8G  384K  7.8G   1% /dev/shm
tmpfs                     3.1G  286M  2.9G  10% /run
tmpfs                     4.0M     0  4.0M   0% /sys/fs/cgroup
/dev/mapper/rootvg-root   2.0G  815M  1.1G  45% /
/dev/mapper/rootvg-usr    6.8G  3.1G  3.4G  47% /usr
/dev/mapper/rootvg-opt     60G  3.3G   55G   6% /opt
/dev/mapper/rootvg-var    2.0G  608M  1.3G  33% /var
/dev/sda1                 986M  162M  774M  18% /boot
/dev/mapper/rootvg-home   2.0G  131M  1.7G   8% /home
/dev/mapper/rootvg-tmp    2.4G  1.8G  531M  78% /tmp
/dev/mapper/rootvg-home2  2.0G   72M  1.8G   4% /home2
tmpfs                     1.6G     0  1.6G   0% /run/user/102129
```

We used SUSE Linux 15.3 for our OS.

### Network
We requested the following port configuration from the firewall team. This config reflects that we will serve cBioPortal on port 8080 and, in the future, Keycloak on port 8180.
```
Source Server	                Port	                Target Server	Service	Protocol
10.16.103.99  (lipcbioap01)	22,80,8080,8180,443	10.250.0.0/16	http	TCP/IP
10.16.103.99  (lipcbioap01)	22,80,8080,8180,443	10.1.0.0/16	http	TCP/IP
10.16.103.99  (lipcbioap01)	22,80,8080,8180,443	10.44.202.1/16	http	TCP/IP
```

The target server ranges correspond to workstations in various parts of MDL:
`10.250.0.0/16` are VPN users.
`10.1.0.0/16` are machines in A6-214.
`10.44.202.1/16	` are machines in Cyriac's office.

### Docker
By default, Docker `20.10.17-ce` stores image data to `/var/lib/`. In our deployment, we store this data in `/opt/` and use `/var/` strictly for logs.

First, ensure all docker services are stopped.
```bash
sudo systemctl stop docker
sudo systemctl stop docker.service
sudo systemctl stop docker.socket
```

Then move the docker root to `/opt`
```bash
sudo mv /var/lib/docker /opt
```

Next, add a line to `/etc/docker/daemon.json` to tell the docker daemon the new data root location.
```json
{
  "data-root": "/opt/docker"
}
```

Finally, start the Docker service and ensure no errors occur.

```bash
sudo systemctl start docker
```

### Certificate
To utilize https, we require a certificate signed by a Certificate Authority (CA). 

Start with creating a new directory private key, called `example.key` in our case. This will prompt you for a password.

```bash
openssl genrsa -des3 -out example.key 2048
```

After the key is successfully created, save the password you used to a `.pass` file:

```bash
echo <YOUR_PASSWORD> >> example.pass
```

Next, generate a Certificate Signing Request (CSR). It will prompt you for information about your organization, email, etc. **ISS requires that you use your FQDN for the Common Name**.

```bash
openssl req -new -out example.csr
```

This CSR can then be given to a Certificate Authority (CA), who will sign it and give you a certificate file (`.cer` or `.crt`) that `nginx` can use. 

**NOTE:** For certs that we have received from UCLA, the cert chain is improperly ordered, causing `nginx` to throw  `error:0B080074:x509 certificate routines:X509_check_private_key:key values mismatch`. You can check if this is the case by observing the `.cer` file with `openssl x509 -noout -text -in example.cer`. The subject line should match your distuingished name from the previous step. If it looks like information from the CA, **move the last certificate in the chain to the beginning.**

### Nginx
First, start the `nginx` service:
```bash!
sudo systemctl start nginx
```
and verify that it is listening on port 80:
```bash
sudo lsof -i:80
```

Add the following to your `nginx.conf`, again replacing bracketed values with your own. In our case, this file was located at `/etc/nginx/nginx.conf`.

```
http {
    ...
    server {
        listen       80;
        listen       443 ssl;
        server_name  <your DNS 1> <your DNS 2>;
        ssl_certificate </path/to/your/example.cer>;
        ssl_certificate_key </path/to/your/example.key>;
        ssl_password_file </path/to/your/example.pass>;
        ...
        location / {
            proxy_set_header   X-Forwarded-For $remote_addr;
            proxy_set_header   Host $http_host;
            proxy_pass "http://127.0.0.1:8080";
        }
    ...
```

Finally, restart nginx:
```bash
sudo systemctl restart nginx
```

If this command succeeds, your web app should now support https.

### Keytool
[keytool](https://docs.oracle.com/javase/8/docs/technotes/tools/windows/keytool.html) is a Java tool for creating a key that our application will use as a signing certificate.

First, install a Java Runtime Environment (JRE) if you do not already have a Java installation. You can also use a JDK, but we only require a JRE.

```bash
sudo zypper install default-jre
```

Then, create a keystore using `keytool`.

```bash
keytool -genkey -alias secure-key -keyalg RSA -keystore samlKeystore.jks
```
This will create a Java keystore for a key called `secure-key` and place the keystore in a file named `samlKeystore.jks`. You will be prompted for:

- keystore password (required, for example: `apollo1`)
- your name, organization and location (optional)
- key password for secure-key (required, for example `apollo2`)


## SAML Authentication
<!-- ### Keycloak
Keycloak is a tool supported by cBioPortal that manages authorization, authentication, and user management. Keycloak also supports external Identity Providers (IDP), such as Okta (used by UCLA), using them to provide users and authorization levels.

Keycloak deploys as a server and a MySQL database, both in Docker containers.

First, set up a local network for Keycloak to communciate with its database:

```bash!
docker network create kcnet
```
Then, start the database container:
```bash!
docker run -d --restart=always \
    --name=kcdb \
    --net=kcnet \
    -v "/home/svcpathx/cbio/kcdb-files:/var/lib/mysql" \
    -e MYSQL_DATABASE=keycloak \
    -e MYSQL_USER=keycloak \
    -e MYSQL_PASSWORD=password \
    -e MYSQL_ROOT_PASSWORD=root_password \
    mysql:5.7
```

Finally, we'll start the Keycloak server itself. Note we run this on port 8180, and 

**WARNING:** The Keycloak server will be visible on your network. Choose a strong admin password.

```bash!
docker run -d --restart=always \
    --name=cbiokc \
    --net=kcnet \
    -p 8180:8080 \
    -e DB_VENDOR=mysql \
    -e DB_ADDR=kcdb \
    -e KEYCLOAK_USER=admin \
    -e "KEYCLOAK_PASSWORD=root_password" \
    -e KC_SPI_TRUSTSTORE_FILE_FILE=samlKeystore.jks \
    -e "KC_SPI_TRUSTSTORE_FILE_PASSWORD=apollo1" \
    -e KC_SPI_TRUSTSTORE_FILE_HOSTNAME_VERIFICATION_POLICY=ANY \
    -v /home/svcpathx/cbio/saml/samlKeystore.jks:/cbioportal-webapp/WEB-INF/classes/samlKeystore.jks \
    jboss/keycloak:4.8.3.Final
```
Then, create the MySQL database that Keycloak will use:

**NOTE:** The directory `kcdb-files` will be where Keycloak stores its data. In our case, we used `/home/svcpathx/cbio/kcdb-files/`.

```bash
docker run -d --restart=always \
    --name=kcdb \
    --net=kcnet \
    -v "/home/svcpathx/cbio/kcdb-files:/var/lib/mysql" \
    -e MYSQL_DATABASE=keycloak \
    -e MYSQL_USER=keycloak \
    -e MYSQL_PASSWORD=password \
    -e MYSQL_ROOT_PASSWORD=root_password \
    mysql:5.7
```
 -->
### cBioPortal Configuration
Modify `config/portal.properties` to reflect the following settings:

```
saml.sp.metadata.entitybaseurl=https://cbio.mednet.ucla.edu:443
saml.idp.metadata.location=classpath:/user-tailored-metadata.xml
saml.idp.metadata.entityid=https://<YOUR_FQDN>:8180/auth/realms/cbioportal
saml.keystore.location=classpath:/samlKeystore.jks
saml.keystore.password=apollo1
saml.keystore.private-key.key=secure-key
saml.keystore.private-key.password=apollo2
saml.keystore.default-key=secure-key
saml.custom.userservice.class=org.cbioportal.security.spring.authentication.keycloak.SAMLUserDetailsServiceImpl
saml.logout.local=false
saml.logout.url=/
```

Next, modify `docker-compose.yaml` as follows. Leave the existing volumes as they are.

The only change to `command:` is setting `-Dauthenticate=saml` and `--proxy-base-url` to our FQDN

```
services:
    cbioportal:
        volumes:
         - ...
         - ./samlKeystore.jks:/cbioportal-webapp/WEB-INF/classes/samlKeystore.jks
         - ./client-tailored-saml-idp-metadata.xml:/cbioportal-webapp/WEB-INF/classes/client-tailored-saml-idp-metadata.xml
        ...
        command: /bin/sh -c "java -Xms2g -Xmx4g -Dauthenticate=saml -Dsession.service.url=http://cbioportal-session:5000/api/sessions/my_portal/ --proxy-base-url https://cbio.mednet.ucla.edu -jar webapp-runner.jar -AmaxHttpHeaderSize=16384 -AconnectionTimeout=20000 --enable-compression /cbioportal-webapp"
```

### Database Configuration
Use `docker ps` to see the name of the container for your MySQL database. In this case, it's `7e85dd90fa56_cbioportal-database-container`

![](https://i.imgur.com/NuPxqTZ.png)

Now, log in to the MySQL shell for that database. (Replace the example name with the name of your database from the previous step).

```bash
cd bioportal-docker-compose/
mysql -u cbio_user 7e85dd90fa56_cbioportal-database-container
```

By default, the login for this account is as follows:
```
username: cbio_user
password: somepassword
```

See the available databases by running `SHOW DATABASES;` at the MySQL prompt.

![](https://i.imgur.com/tm7tMy7.png)

Connect to the `cbioportal` database:
```
USE cbioportal;
```

Now, execute the following commands to create an admin user that has authority to access all studies.

**NOTE**: Replace the example email and username with your desired credentials. Ensure that the email matches exactly in both entries.

```SQL
INSERT INTO cbioportal.users (EMAIL, NAME, ENABLED) 
VALUES ('iatol@mednet.ucla.edu', 'Ian', 1);
```

```SQL
INSERT INTO cbioportal.authorities (EMAIL, AUTHORITY)
VALUES ('iatol@mednet.ucla.edu', 'cbioportal:ALL');
```