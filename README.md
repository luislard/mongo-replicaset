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

Next we need to create the **key file**.

> The contents of the keyfile serves as the shared password for the members of the replica set. The content of the keyfile must be the same for all members of the replica set.