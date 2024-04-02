# DevOps Challenge

This repository contains the (partial) solution to [this Doodle DevOps challenge](https://github.com/DoodleScheduling/hiring-challenges/tree/master/devops-engineer).

## Contents

The repository is structured into four sub folders: 
- configmaps
- manifests
- secrets
- storage

The manifests folder holds the manifest files deploying Keycloak, Infinispan and Postgresql.
Each file contains the service, deployment and any persistent volume claims that deployment may need.

Config maps and the source files for the config map data are found in the configmaps folder.

The credentials for Keycloak, Infinispan and Postgresql are stored in secret manifests in the secrets folder.

Storage class and Persistent Volume manifests are stored in the storage folder.

## How to run

To deploy the solution on a local kubernetes engine it should be possible to run the command:

```bash
kubectl apply -f . --recursive
```

from the project root folder.

Before deploying it might be necessary to change the host path and node affinity of the persistent volume in storage/pv.yaml to something suitable.

## Aproach

I have chosen to deploy on Docker Desktop Kubernetes and use Postgresql as database for Keycloak.
Since I have never used a templating tool like Helm or Kustomization, I decided to go for vanilla kubernetes.

I run Keycloak in developer mode to avoid having to use TLS for this exercise.

This turned out to be a bigger challenge than first anticipated, with a lot of road bumps along the way.

### Architecture

There are three main deployments:
- Keycloak
- Infinispan
- Postgres


#### Keycloak

Keycloak has a Service of type Loadbalancer that exposes the keycloak console on localhost:8080.

##### Database
It is connected to a database through the environment variables:

```yaml
- name: KC_DB
  value: postgres
- name: KC_DB_URL_HOST
  value: postgres
- name: KC_DB_URL_DATABASE
  value: keycloak_db
- name: KC_DB_USERNAME
  valueFrom:
      secretKeyRef:
        name: postgres-credentials
        key: username
- name: KC_DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: postgres-credentials
      key: password
```

The function of the KC environment variables can be found [here](https://www.keycloak.org/server/all-config).

The KC_DB_URL_HOST points to a service named postgres.
The KC_DB_USERNAME and KC_DB_PASSWORD are retrieved from the postgres-credentials secret.

##### Remote Infinispan discovery

Keycloak has an embeded infinispan subsystem.

To enable Keycloak to connect to an external Infinispan service the following environmental variables have to be set:

```yaml
# Turn on infinispan remote cache and service discovery in developer mode
- name: KC_CACHE
  value: ispn
  # Set the transport layer stack to kubernetes
- name: KC_CACHE_STACK
  value: kubernetes
  # set the dns.query to target the service name infinispan, to allow service discovery
- name: JAVA_OPTS_APPEND
  value: "-Djgroups.dns.query=infinispan.default.svc.cluster.local"

```

KC_CACHE_STACK tells Keycloak to use a communication stack suitable for kubernetes. This enables node discovery on kubernetes.

To perform node discovery the Jgroups will query the infinispan service. Which requires the JAVA_OPTS_APPEND variable to be set.

Since Keycloak is deployed in developer mode, KC_CACHE must be set to ispn. Otherwise keycloak would use a local cache distribution and ignore the other settings.

##### Configuring Remote Store Caches

Caches are configured in the /opt/keycloak/conf/cache-ispn.xml in the Keycloak container.

In the cache container section of the configuration file, caches are configured as either local or distributed. To make a distributed cache connect to a remote Infinispan the remote store element has to be added as shown below.

```xml
<distributed-cache name="sessions" owners="1">
         <persistence passivation="false">
            <remote-store xmlns="urn:infinispan:config:store:remote:14.0" cache="sessions" purge="false" preload="false" segmented="false" shared="true" raw-values="true" >
                <remote-server host="${env.KEYCLOAK_CACHE_REMOTE_HOST}" port="11222" />
                <security>
                   <authentication server-name="${env.KEYCLOAK_CACHE_REMOTE_HOST}">
                      <digest username="${env.KEYCLOAK_CACHE_REMOTE_USERNAME}" password="${env.KEYCLOAK_CACHE_REMOTE_PASSWORD}" realm="default" />
                   </authentication>
                </security>
                 <!-- <property name="marshaller">org.keycloak.cluster.infinispan.KeycloakHotRodMarshallerFactory</property> -->
             </remote-store>
         </persistence>
</distributed-cache>
```
To authenticate with the remote Infinispan one option is to use a username and password digest as shown above.

The KEYCLOAK_CACHE_REMOTE_USERNAME, KEYCLOAK_CACHE_REMOTE_PASSWORD and KEYCLOAK_CACHE_REMOTE_HOST environmental variables are defined in the Keycloak Deployment yaml. 


To mount the changed ispn-cache.xml, a configMap was created from the modified file. The configmap is then mounted on as a volume at the path of the original ispn-cache.xml. The subPath declaration ensures that the value of the config map key kc-ispn.xml is the data mounted at the path. It is worth noting that configMaps mounted with subPath do not update in the container when changed. Instead a restart of the container is required.

```yaml
volumeMounts:
  - name: "keycloak-config-volume"
    mountPath: "/opt/keycloak/conf/cache-ispn.xml"
    subPath: "kc-ispn.xml"
```

##### Arguments

```yaml
args: ["start-dev", "--cache-config-file=cache-ispn.xml", "--spi-connections-infinispan-quarkus-site-name='keycloak'"]

```

The start-dev argument is used to avoid having to deal with TLS for this challenge.

The --spi-connections-infinispan-quarkus-site-name flag is used because of the following documentation from [Keycloak](https://www.keycloak.org/high-availability/connect-keycloak-to-external-infinispan#:~:text=The%20spi%2Dconnections%2Dinfinispan%2D,from%20the%20external%20Infinispan%20deployment).

`The spi-connections-infinispan-quarkus-site-name is an arbitrary Infinispan site name which Keycloak needs for its embedded Infinispan deployment when a remote store is used`

#### Infinispan

Infinispan version 14.0 is used, as 15.0 had a different jgroup version than Keycloak and therefore ignored packets sent from Keycloak.

Infinispan is setup with a Loadbalancer type Service to enable accesss to the infinispan console on localhost:11222

The service also exposes port 7800 which is the port for intercommunication between infinispan systems.

##### Remote Infinispan discovery

To enable the Remote infinispan the communicate and discover other nodes, just as in the case of Keycloak, the communication stack and jgroup dns query had to be set.

```yaml
- name: JAVA_OPTIONS
  value: "-Dinfinispan.cluster.stack=kubernetes -Djgroups.dns.query=infinispan.default.svc.cluster.local -Dinfinispan.cluster.name=ISPN"
```

-Dinfinispan.cluster.name is set to ensure the same jgroups channel name as is set in Keycloak.


##### Configuring Remote Store Caches

To enable setup the caches that Keycloak will attempt to connect with in the external infinispan, the /opt/infinispan/server/conf/infinispan.xml needs to be modified.

Adding the replicated and distributed caches under the cache-container element:

```xml

<cache-container name="default" statistics="true">

    <!-- Replicated cache -->
  <replicated-cache name="work">
      <!-- <encoding media-type="application/x-jboss-marshalling" /> -->
  </replicated-cache>

    <!-- Distributed caches -->
  <distributed-cache name="sessions">
      <!-- <encoding media-type="application/x-jboss-marshalling" /> -->
  </distributed-cache>
```

The modified infinispan.xml was mounted through a configMap just like the Keycloak ispn-cache.xml file.

#### Postgres

Postgres is setup to persist data with a Persistant Volume Claim that is mounted at /var/lib/postgresql/data in the postgres container.

The PVC requests storage of class local-storage. This storage class is declared in storage/pv.yaml.

A persistent volume matching the requested storage and storage class is also defined in the storage/pv.yaml

It uses hostpath to store data locally on the host machine. It also uses:

```yaml
 nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          - docker-desktop
```

to enable deployment in docker-desktop kubernetes.

## Challenges

It was quite a challenge going through this exercise.
I spent the first 8 hours setting up the three deployments and configuring persistence for postgres.

Getting Keycloak to connect with the remote Infinispan was the real challenge. And it seems I am not the only one that has struggled with this on the internet :)

At first I had trouble just getting the Keycloak and Infinispan to discover each other. It was clear that multiple infinispan instances automatically discovered each other, but I spent a lot of time trying to get keycloak to find Infinispan.

The keycloak documentation is sparse, and doesn't seem to mention that while you need to confiugre keycloak to use the kubernetes communication stack, you also need to configure something similar for infinispan.

Once I had jumped that hurdle I, saw error messages in the infinispan logs, saying that it was disregarding packets from Keycloak because the jgroups version was different. I then had to downgrade infinispan from 15.0 to 14.0 to go from packet version 5.3.x to 5.2.x which keycloak used.

Serialization and encoding. I tried using jboss-marshalling to tackle encoding and serialization, and it seemed to work, except that it would throw errors if I tried to query in the infinispan console. Stack overflow suggested to turn off encoding. Which seems to work, but supposedly is also less efficient.

Below are a selection of the resources i used to learn how to solve the problems.

Resources:
https://www.keycloak.org/server/caching

https://infinispan.org/docs/stable/titles/configuring/configuring.html#remote-cache-store_persistence

https://engineering.vendavo.com/keycloak-deployment-with-distributed-cache-43f72995674d

https://github.com/keycloak/keycloak/discussions/21890

https://github.com/keycloak/keycloak/issues/26064

https://www.keycloak.org/high-availability/connect-keycloak-to-external-infinispan#:~:text=The%20spi%2Dconnections%2Dinfinispan%2D,from%20the%20external%20Infinispan%20deployment.

https://stackoverflow.com/questions/75833588/keycloak-and-external-infinispan-and-console-throws-error-while-marshaling
 

