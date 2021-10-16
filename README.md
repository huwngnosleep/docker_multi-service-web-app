# Assignment: Create A Multi-Service Multi-Node Web App

## Goal: create networks, volumes, and services for a web-based "cats vs. dogs" voting app.
Here is a basic diagram of how the 5 services will work:

![diagram](./architecture.png)
- All images are on Docker Hub, so you should use editor to craft your commands locally, then paste them into swarm shell (at least that's how I'd do it)
- a `backend` and `frontend` overlay network are needed. Nothing different about them other than that backend will help protect database from the voting web app. (similar to how a VLAN setup might be in traditional architecture)
- The database server should use a named volume for preserving data. Use the new `--mount` format to do this: `--mount type=volume,source=db-data,target=/var/lib/postgresql/data`

### Services (names below should be service names)
- vote
    - bretfisher/examplevotingapp_vote
    - web front end for users to vote dog/cat
    - ideally published on TCP 80. Container listens on 80
    - on frontend network
    - 2+ replicas of this container

- redis
    - redis:3.2
    - key/value storage for incoming votes
    - no public ports
    - on frontend network
    - 1 replica NOTE VIDEO SAYS TWO BUT ONLY ONE NEEDED

- worker
    - bretfisher/examplevotingapp_worker
    - backend processor of redis and storing results in postgres
    - no public ports
    - on frontend and backend networks
    - 1 replica

- db
    - postgres:9.4
    - one named volume needed, pointing to /var/lib/postgresql/data
    - on backend network
    - 1 replica
    - remember set env for password-less connections -e POSTGRES_HOST_AUTH_METHOD=trust

- result
    - bretfisher/examplevotingapp_result
    - web app that shows results
    - runs on high port since just for admins (lets imagine)
    - so run on a high port of your choosing (I choose 5001), container listens on 80
    - on backend network
    - 1 replica


### Answer

###### Create 3-node swarm cluster
`docker-machine create --driver virtualbox host1`
`docker-machine create --driver virtualbox host2`
`docker-machine create --driver virtualbox host3`

###### Create Swarm from host1
`docker-machine ssh host1`
`docker swarm init --advertise-addr 192.168.99.100`

###### Join two others to cluster
`docker swarm join --token <swarm-token> <host-IP>`

###### Create network: front-end network and back-end network
`docker network create -d overlay frontend`
`docker network create -d overlay backend`

###### Create services
`docker service create --name vote -p 80:80 --network frontend --replicas 2 bretfisher/examplevotingapp_vote`
`docker service create --name redis --network frontend redis:3.2`
`docker service create --name db --network backend -e POSTGRES_HOST_AUTH_METHOD=trust --mount type=volume,source=db-data,target=/var/lib/postgresql/data postgres:9.4`
`docker service create --name worker --network frontend --network backend bretfisher/examplevotingapp_worker`
`docker service create --name result --network backend -p 5001:80 bretfisher/examplevotingapp_result`

###### View result on 192.168.99.100
![result](result.png)