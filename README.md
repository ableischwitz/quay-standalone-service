# quay-standalone-service
Configuration and SystemD files to run Quay-3.4+ on a single host

# Precautions/Introduction

Please read the [official documentation of Red Hat Quay](https://access.redhat.com/documentation/en-us/red_hat_quay/3.4/html/deploy_red_hat_quay_for_proof-of-concept_non-production_purposes/index) and validate the steps described here.
In doubts the official documentation would be source of truth.

Please also note that this kind of setup is only for prove of concept setups and high-availability requirements are not fulfilled by this.

# Preparations/Assumptions

* This setup assumes that local storage will be used - even though a recommended S3 setup would be applicable by changing the configuration of Quay and removing the additional volume of the `quay-service.service` systemd unit file.

* Prior starting up **Quay** service, the database has to be created and prepared.

## Prepare filesystems

* The following directories are required:
  * /opt/quay/config
  * /opt/quay/db
  * /opt/quay/redis
  * /opt/quay/storage
  * /opt/clair/config

It's best practice to separate at least `/opt/quay/storage` and `/opt/quay/db` as those directories will tend to grow.

```
# Add proper SELinux types for container filesystem
% semanage fcontext -a -t container_file_t '/opt/quay/(config|storage|redis)(/.*)?'
% semanage fcontext -a -t container_file_t '/opt/clair/config(/.*)?'


# Create filesystem structure for Quay/Clair
% mkdir -p /opt/quay/{config,db,storage,redis}
% mkdir -p /opt/clair/config

# Adjust permissions for postgresql
% setfacl -m u:26:-wx /opt/quay/db
```

## Prepare firewall

```
# Open ports for Web-UI
% firewall-cmd --permanent --zone=public --add-service http
% firewall-cmd --permanent --zone=public --add-service https

# Open port for Clair-v4
% firewall-cmd --permanent --zone=public --add-port=8080/tcp

# Open port for PostgreSQL - this will expose the database outside the host!!
% firewall-cmd --permanent --zone=public --add-service postgresql

# Open port for Redis - same for redis
% firewall-cmd --permanent --zone=public --add-service redis

# Apply configuration
% firewall-cmd --reload
```

## Provide pull-secret to download container images

The SystemD unit-files will make use of `/etc/container/pull-secret.json` which needs to be created prior starting the services.
Even though one may pull the required images manually and have SystemD simply start the services, its easier in case of updated versions to only change a variable and restart the service.
In case the required image-versions are already present on the system, the **pull-secret.json** may not be needed.

## Adjust configuration options

Most configuration can be done by changing the variables in `/etc/sysconfig/quay`, but some may also require adjustements at other locations.
It's always a good advise to change configuration settings for Quay by using the [configuration tool](#start-quay-configuration-container):

**Note:** remember to start **PostgreSQL** and **Redis** before starting the configuration container.

## Start Redis and Postgresql

After `pull-secret.json` was configured or the required container have been pulled, it's time to start the database and also Redis.

```
% systemctl start quay-redis
% systemctl start quay-database
```

Check for errors during start:
```
% journalctl -xf
% podman logs quay-database
% podman logs quay-redis
```

## Configure Quay database

After PostgreSQL started up, it's time to adjust some configuration for the quay and clair database:

```
##### Start psql session in quay-database container
% podman exec -it quay-database /usr/bin/psql
psql (10.15)
Type "help" for help.

##### Create quay database
postgres=# CREATE DATABASE quaydb;
CREATE DATABASE

##### Switch to quay database
postgresql=# \c quaydb

##### Create pg_trgm extension for quay database
quaydb=# CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION

##### Create quay database user and grant permissions
quaydb=# CREATE USER quayuser WITH PASSWORD 'quaypass' CREATDB;
CREATE USER
quaydb=# GRANT ALL PRIVILEGES ON DATABASE quaydb to quayuser;
GRANT
<ctrl+d>
```

## Configure Clair database

```
##### Start psql session in quay-database container
% podman exec -it quay-database /usr/bin/psql
psql (10.15)
Type "help" for help.

##### Create clair database
postgres=# CREATE DATABASE clair;
CREATE DATABASE

##### Switch to clair database
postgres=# \c clair

##### Create uuid-ossp extension for clair database
clair=# CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION

##### Create clair database user and grant permissions
clair=# create USER clairuser WITH PASSWORD 'clairpass' CREATEDB;
CREATE USER
clair=# GRANT ALL PRIVILEGES ON DATABASE clair TO clairuser;
GRANT
<ctrl+d>
```

## Check database connection

```
##### on Quay/Clair machine
% sudo yum module enable postgresql:10/client
% sudo yum -y install postgresql
....

% psql -h localhost -d quaydb -U quayuser
Password for user quayuser: ******   (quaypass)
psql (10.15)
Type "help" for help.

quaydb=> \q


% psql -h localhost -d clair -U clairuser
Password for user clairuser: ***** (clairpass)
psql (10.15)
Type "help" for help.

clair=> \q
```

## Configure Clair

By default Clair will use `/opt/clair/config/config.yaml` for it's configuration. Manual changes need to be applied to ensure proper database connection strings are used.
Adjust `/opt/clair/config/config-example.yaml` and copy/rename it to `config.yaml` prior staring Clair.
The host `quay.example.com` will stand for the postgresql-hostname, but as this is an all-in-one setup, they should be the same.

In case `Quay` will use custom signed certificates, the CA-certificates used for signing should be located in `/opt/clair/config/extra_ca_certs`.

## Start Quay-configuration container

```
% podman run --rm -it --name quay_config -p 8080:8080 -v /opt/quay/config/:/conf/stack:Z registry.redhat.io/quay/quay-rhel8:v3.4.1 config admin
```

# Starting services

With the configuration done, it's now time to (re)-start the services

```
## nice thing about having all quay-related service starting with `quay` is that they may all be stopped at once:
% systemctl stop quay\*

## Start Quay
% systemctl start quay-service

## Start Clair scanning
% systemctl start quay-clair-service

## Start Quay-mirror #1
% systemctl start quay-mirror@1
```

In case more mirror workers would be required, just count them up `quay-mirror@1`, `quay-mirror@2` ... `quay-mirror@n`. Keep in mind that they all will require some resources.

By default all container should provide a status for SystemD, so that `systemctl status quay-\*` should provide a good overview of the services. If some closer look at the processes inside the container
would be needed, use `podman logs -f quay-service` or any other container instead of **quay-service**.

SystemD will also make sure that containers will get restarted in case of a failure - so if a container is stopped via `podman`, SystemD will restart it.

`quay-clair-{indexer,matcher,notifier}` are SystemD services which need to get some adjustments and are for a separated Clair setup on wich all services are running in their own container (with own IP/port).
