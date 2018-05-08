MONGO REPLICASET WITH DOCKER
============================


Create volume

```bash
$ docker volume create --name mongo_storage
```

Attach your first container to the volume 

```bash
$ docker run --name mongoNode1 \
    -v mongo_storage:/data \
    -d mongo \
    --smallfiles
```

**Note:**
> **--smallfiles** 
> Sets MongoDB to use a smaller default file size. The --smallfiles option reduces the initial size for data files and > limits the maximum size to 512 megabytes. --smallfiles also reduces the size of each journal file from 1 gigabyte to > 128 megabytes. Use --smallfiles if you have a large number of databases that each holds a small quantity of data.
> 
> The --smallfiles option can lead the mongod instance to create a large number of files, which can affect performance > for larger databases.



Next we need to create the **key file**.

> The contents of the keyfile serves as the shared password for the members of the replica set. The content of the keyfile must be the same for all members of the replica set.

```bash
$ openssl rand -base64 741 > mongo-keyfile
$ chmod 600 mongo-keyfile
```

Next let’s create the folders where is going to hold the data, keyfile and configurations inside the **mongo_storage** volume:

```bash
$ docker exec mongoNode1 bash -c 'mkdir /data/keyfile /data/admin'
```

The next step is to create some admin users, let’s create a **admin.js** and a **replica.js** file that looks like this:

This file creates 2 users on db admin:
* One user with userAdminAnyDatabase Role.
* One user with clusterAdmin Role.

```javascript
// admin.js
admin = db.getSiblingDB("admin")
// creation of the admin user
admin.createUser(
  {
    user: "cristian",
    pwd: "cristianPassword2017",
    roles: [ { role: "userAdminAnyDatabase", db: "admin" } ]
  }
)
// let's authenticate to create the other user
db.getSiblingDB("admin").auth("cristian", "cristianPassword2017" )
// creation of the replica set admin user
db.getSiblingDB("admin").createUser(
  {
    "user" : "replicaAdmin",
    "pwd" : "replicaAdminPassword2017",
    roles: [ { "role" : "clusterAdmin", "db" : "admin" } ]
  }
)
```

```javascript
//replica.js
rs.initiate({
 _id: 'rs1',
 members: [{
  _id: 0, host: 'manager1:27017'
 }]
})
```

> Passwords should be random, long, and complex to ensure system security and to prevent or delay malicious access. See Database User Roles for a full list of [built-in roles](https://docs.mongodb.com/manual/reference/built-in-roles/#database-user-roles) and related to database administration operations.


What we have done until know:

* created the **mongo_storage**, docker volume.
* created the **mongo-keyfile**, openssl key generation.
* created the **admin.js file**, admin users for mongoDB.
* created the **replica.js file**, to init the replica set.

Ok let’s continue with passing the files to the container.

```bash
$ docker cp admin.js mongoNode1:/data/admin/
$ docker cp replica.js mongoNode1:/data/admin/
$ docker cp mongo-keyfile mongoNode1:/data/keyfile/
```

```bash
// change folder owner to the user container
$ docker exec mongoNode1 bash -c 'chown -R mongodb:mongodb /data'
```

What we have done is that we pass the files needed to the container, and then **change the /data folder owner to the container user**, since the the container user is the user that will need access to this folder and files.

Now everything has been set, and we are ready to restart the mongod instance with the replica set configurations.

Before we start the authenticated mongo container let’s create an env file to set our users and passwords.

```bash
# .env file
MONGO_USER_ADMIN=cristian
MONGO_PASS_ADMIN=cristianPassword2017

MONGO_REPLICA_ADMIN=replicaAdmin
MONGO_PASS_REPLICA=replicaAdminPassword2017
```

Now we need to remove the container and start a new one. Why ?, because we need to provide the replica set and authentication parameters, and to do that we need to run the following command:

```bash
# first let's remove our container
$ docker rm -f mongoNode1
// now lets start our container with authentication 
$ docker run --name mongoNode1 --hostname mongoNode1 \
-v mongo_storage:/data \
--env-file env \
--add-host manager1:192.168.99.100 \
--add-host worker1:192.168.99.101 \
--add-host worker2:192.168.99.102 \
-p 27017:27017 \
-d mongo --smallfiles \
--keyFile /data/keyfile/mongo-keyfile \
--replSet 'rs1' \
--storageEngine wiredTiger \
--port 27017
```