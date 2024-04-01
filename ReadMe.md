# DevOps Challenge

This repository contains the (partial) solution to [this Doodle DevOps challenge](https://github.com/DoodleScheduling/hiring-challenges/tree/master/devops-engineer).

## Contents

Explain directory and files.


## Aproach

Docker Desktop Kubernetes.

Deploy Keycloak, Infinispan, Postgres, Connect Keycloak and Postgres and test persistance, by deleting the container.

Use Persistent Volume Claim and Persistent Volume with local storage class.



## Challenges

Persistent Volume

Infinispan remote cache.

Resources:
https://www.keycloak.org/server/caching

https://infinispan.org/docs/stable/titles/configuring/configuring.html#remote-cache-store_persistence

https://engineering.vendavo.com/keycloak-deployment-with-distributed-cache-43f72995674d

https://github.com/keycloak/keycloak/discussions/21890

https://github.com/keycloak/keycloak/issues/26064

https://www.keycloak.org/high-availability/connect-keycloak-to-external-infinispan#:~:text=The%20spi%2Dconnections%2Dinfinispan%2D,from%20the%20external%20Infinispan%20deployment.

https://stackoverflow.com/questions/75833588/keycloak-and-external-infinispan-and-console-throws-error-while-marshaling

## Further Work

Secrets and Config Maps



Build Keycloak image with config file.
Build infinispan image with config file.

Try exposing keycloak port 11222