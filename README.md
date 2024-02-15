CSS: myStyle.css
<link rel="stylesheet" href="myStyle.css" />

# Concierge Debit Accounts Service

Concierge Debit Accounts Service (CDAS) is a collection of endpoints to manage debit accounts and all movements related to specific account.

## Features

- Create new debit accounts
- Close debit accounts
- Close source accounts and transfer founds to new debit account
- Register debit account movements

## Environment Configuration

For localhost, the recommendation is to create following components:

| Component             | Details                           |
|-----------------------|-----------------------------------|
| Postgres v14          | A one instance in read-write mode |
| Keycloak v16.1.1      | Auth Server with OpenID Connect   |
| Eureka Server         | Discovery Server as a Service     |
| API Gateway           | API Gateway Server as a Service   |
| Debit Account Service | This code as a Service            |

## Configuring Debit Account service in localhost

>**Pre-requisites:**
>
>`Docker` is required for almost all configurations in localhost, please make sure you have installed and configured
> Docker in your local environment.

## _1a) (Recommended) Database with only one readWrite instance:_
### Running Debit Account service using only one database.

Execute in localhost following command:
```sh
docker network create concierge
docker volume create db-server-vol
docker run -d --rm --name postgres \
-e POSTGRES_PASSWORD=postgres \
-e POSTGRES_USER=postgres \
-e POSTGRES_DB=concierge-debit-accounts  \
-v db-server-vol:/var/lib/postgresql/data \
-p 5432:5432 \
--network concierge \
postgres:14-alpine
```

>**IMPORTANT:**
>
>  The Concierge Debit Account Service (CDAS) will require following data configured in your IDE as Java Arguments with the above configuration:
>
> ```sh
> -DDEBIT_ACCOUNTS_DB_USER=postgres
> -DDEBIT_ACCOUNTS_DB_PASSWORD=postgres
> -DDEBIT_ACCOUNTS_DB_NAME=concierge-debit-accounts
> -DDEBIT_ACCOUNTS_DB_READ_WRITE_HOST=localhost
> -DDEBIT_ACCOUNTS_DB_READ_WRITE_PORT=5432
> -DDEBIT_ACCOUNTS_DB_READ_ONLY_HOST=localhost
> -DDEBIT_ACCOUNTS_DB_READ_ONLY_PORT=5432
> ```

## _1b) Database with readOnly and readWrite instances:_
### Running Debit Account service using Read- Write and Read-Only database instances.
>**Note:**
>
>This is a preferred option over  the option 1a because this configuration allows to
developers verify the use of transactional scopes based on the need of the query.

>**IMPORTANT:**
>
> Using docker to build master-replica mode fails sometimes on starting replica instance, so try to configure
> the database externally as mater-replica mode or use a Cloud service with same configuration.

>**Note:**
>- **master** instance is a **Read-Write** instance, for this document master will be a synonymous for
>**postgres-db-master-cda**.
>- **replica** instance is a **Read-Only** instance, for this document master will be a synonymous for
>**postgres-db-replica-cda**.

Execute in localhost following commands:
```sh
docker network create concierge
```
```sh
docker run -d --rm --name postgres-db-master-cda -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres \
-e POSTGRES_DB=concierge-debit-accounts -p 5404:5432 --net concierge postgres:14-alpine
```
```sh
docker run -d --rm --name postgres-db-replica-cda -e POSTGRES_PASSWORD=postgres -e POSTGRES_USER=postgres \
-e POSTGRES_DB=concierge-debit-accounts -p 5405:5432 --net concierge postgres:14-alpine
```
Access to **master** instance:
```sh
docker exec -it postgres-db-master-cda bash
```
Open the postgres configuration file:
```sh
vi var/lib/postgresql/data/postgresql.conf
```

>**Find the following keys and set the values under `CUSTOMIZED OPTIONS` sections as follows :**
>- wal_level=replica
>- max_wal_senders=2
>- archive_mod=on
>- archive_command='cp %p /tmp/%f'

Save and close the file.

Open the postgres client authentication configuration file:
```sh
vi var/lib/postgresql/data/pg_hba.conf
```
>**Append following line at the bottom:**
>- host replication all `{DOCKER_CIDR_REPLICA_INSTANCE}` trust

Where
`{DOCKER_CIDR_REPLICA_INSTANCE}` should be replaced with the value retrieved from the following command:

```sh
docker network inspect concierge -f '{{json .Containers}}' | jq '.'
```
>**Note:**
>
>Use the value from **IPv4Address** from **postgres-db-replica-cda** instance.
>
>You should have something like this:
>
>`host replication all 172.18.0.3/16 trust`

>
>**IMPORTANT:**
>
> Make sure you choose network from `postgres-db-replica-cda` if you choose network from  `postgres-db-master-cda`
> the container is going to shut down.

Save and close the file.

Finally, exist from **master** container and restart the instance using this:

```sh
docker restart postgres-db-master-cda
```

Verify that **master** is up and running after restarting:

```sh
docker ps | grep postgres-db-master-cda
```

Access to **replica** instance:
```sh
docker exec -it postgres-db-replica-cda bash
```
Execute following commands:

```sh
su - postgres
rm -rf /var/lib/postgresql/data/* && pg_basebackup -U postgres -R -D /var/lib/postgresql/data/ \
--host={DOCKER_GATEWAY_CONCIERGE_NETWORK} --port={PORT_MASTER}
```
Where:

`{PORT_MASTER}` was defined as **5404** on creating the instance

`{DOCKER_GATEWAY_CONCIERGE_NETWORK}` should be replaced with the value retrieved from the following command:
```sh
docker network inspect concierge -f '{{json .IPAM.Config}}' | jq '.'
```
>**Note:**
>
>Use the value from **Gateway** property.

Open the postgres configuration file:
```sh
vi var/lib/postgresql/data/postgresql.conf
```

>**Find the following keys and set the values as follows:**
>- hot_standby=on

Save and close the file.

Finally, restart **replica** instance using this:

```sh
docker restart postgres-db-replica-cda
```

Verify that **replica** is up and running:

```sh
docker ps | grep postgres-db-replica-cda
```

>**IMPORTANT:**
>
>  The Concierge Debit Account Service (CDAS) will require following data configured in your IDE as Java Arguments with the above configuration:
>
> ```sh
> -DDEBIT_ACCOUNTS_DB_USER=postgres
> -DDEBIT_ACCOUNTS_DB_PASSWORD=postgres
> -DDEBIT_ACCOUNTS_DB_NAME=concierge-debit-accounts
> -DDEBIT_ACCOUNTS_DB_READ_WRITE_HOST=localhost
> -DDEBIT_ACCOUNTS_DB_READ_WRITE_PORT=5404
> -DDEBIT_ACCOUNTS_DB_READ_ONLY_HOST=localhost
> -DDEBIT_ACCOUNTS_DB_READ_ONLY_PORT=5405
> ```


## _2) Run Identity and Access Management Server (Keycloak)_
>**Important:**
> Run following command from concierge-resource location, make sure the location for concierge-realm.json
> file is correct for your local configuration

```sh
docker run -d --rm --name auth-server \
-e KEYCLOAK_USER=admin \
-e KEYCLOAK_PASSWORD=admin \
-e KEYCLOAK_IMPORT=/home/concierge-realm.json \
-e DB_VENDOR=h2 \
-p 8080:8080 \
-v /home/emmanuel/JavaDevelopment/concierge-resources/concierge-realm.json:/home/concierge-realm.json \
jboss/keycloak:16.1.1
```
>**Note:**
>
>Once keycloak is up and running, if you are NOT using docker-compose, one more change is required in order to use
>KeyCloak in localhost, on Admin Console select Realm Settings then Concierge Realm, in General tab, clear the field
>`Frontend Url` or use localhost instead, then save the changes.
>
>The change is because when using docker-compose, `auth-server` is an alias for keycloak, however running debit account
>service from IDE will not find auth-server as hostname from localhost.
>
>It is the same as if you want to run KeyCloak instance with Google or Facebook providers, they will not able to find
> auth-server a valid domain name, in that case, please create a domain name on an external instance like EC2 or GCE
> and make KeyCloak reachable from internet, make the appropriate change to `Frontend Url` field to reflect same domain.


>**Note:**
> To export a realm configuration file from container, run following commands:

Create a real file on container running:
```sh
docker exec -it keycloak /opt/jboss/keycloak/bin/standalone.sh \
-Djboss.socket.binding.port-offset=100 \
-Dkeycloak.migration.action=export \
-Dkeycloak.migration.provider=singleFile \
-Dkeycloak.migration.realmName=concierge \
-Dkeycloak.migration.usersExportStrategy=REALM_FILE \
-Dkeycloak.migration.file=/opt/jboss/keycloak/standalone/data/concierge-realm.json
```

Copy realm file from container to host machine or use volume option from docker:
```sh
docker cp keycloak:/opt/jboss/keycloak/standalone/data/concierge-realm.json ~/concierge-realm.json
```

>**Note:**
The configuration files are examples without warranty, do not use in production environment

## _3) Run Discovery Server (Eureka Server)_
```sh
docker run -d --rm --name=discovery-server \
-p 8018:8018 \
--network concierge \
ngineapps/concierge-discovery-service
```

## _4) Run API Gateway_
```sh
docker run -d --rm --name=gateway-server \
-e DEFAULT_ZONE=http://discovery-server:8018/eureka/ \
-p 8008:8008 \
--network concierge \
ngineapps/concierge-api-gateway
```

## _5) (Optional) Run Prometheus Server_
>**Note:**
Get from Git the prometheus.yaml file before starting prometheus container

>**Note:**
If you are running the concierge debit accounts service from IDE, in your localhost, please add the correct IP:PORT
> to prometheus.yml file, the IP must be the IP Address from physical machine

```sh
docker run -d --name=prometheus -p 9090:9090 \
-v /home/emmanuel/JavaDevelopment/concierge-resources/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus \
--config.file=/etc/prometheus/prometheus.yml
```


## _6) (Optional) Run Grafana_
```sh
docker volume create grafana-storage
docker run -d -p 3000:3000 --name=grafana -v grafana-storage:/var/lib/grafana grafana/grafana
```
>**Note:**
Create a Datasource in grafana to use Prometheus Server

>**Note:**
In Grafana Datasource option, select correct IP inside docker network for prometheus container, use this
> command to get the IP and test in Grafana dashboard

```sh
docker inspect -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' $(docker ps -aqf "name=prometheus")
```

>**Note:**
The above command use prometheus as container name, change it if you named the container with a different name

## _Run Concierge Debit Account Service from IDE_
Please check above sections to set Java arguments for the service.
Add following Java arguments in order to use along with the Discovery Server,
Api Gateway.
The last Java argument is the URL to create and validate JWT Tokens.

```sh
-DDISCOVERY_ENDPOINT=http://localhost:8018/eureka/
-DGATEWAY_URL=http://localhost:8008
-DISSUER_URI=http://localhost:8080/auth/realms/concierge
```

## _Create Concierge Debit Account Service as Docker Image_

Clone the project using following command:
```sh
git clone git@github.com:ngineapps/concierge-accounts.git
```
Create the artifact from root location of the project, at same level as the pom.xml file:
```sh
mvn clean package
```
Create image:
```sh
docker build -t ngineapps/concierge-debit-accounts --no-cache .
```

>**IMPORTANT:**
>
>Once you have the image from the service application, please try to use docker-compose or Kubernetes in case you want to
> run the service from the image instead of trying to run the image in an isolated way.
> The parameterizing can be confusing and error-prone.


***EOF***

