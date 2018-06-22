## Build MongoDB replica set with Docker on local machine


### 1. Use docker-compose to run mongodbs

```
$ docker-compose -f replica-set-compose.yml up
```

check mongodbs is running:

```
$ docker ps
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS              PORTS                                            NAMES
377429727de4        mongo:latest           "docker-entrypoint.s…"   6 seconds ago       Up 5 seconds        0.0.0.0:27017->27017/tcp                         mongo1
8dec66375f49        mongo:latest           "docker-entrypoint.s…"   6 seconds ago       Up 5 seconds        0.0.0.0:27018->27017/tcp                         mongo2
986274ddfc0d        mongo:latest           "docker-entrypoint.s…"   6 seconds ago       Up 5 seconds        0.0.0.0:27019->27017/tcp                         mongo3
```

### 2. Configure replica set

Connect to first mongo:

```
$ docker exec -it mongo1 mongo
```

Configure replica set in mongo shell:

> Find your current IP. ex: 192.168.0.2

```
> rsconf = {
  _id: "rs",
  members: [
    {
     "_id": 0,
     "host": "your_current_ip:27017"
    },
    {
     "_id": 1,
     "host": "your_current_ip:27018"
    },
    {
     "_id": 2,
     "host": "your_current_ip:27019"
    }
   ]
}

> rs.initiate(rsconf)
```

After few seconds, replica set will elect a primary one.

Then check DB status:

```
> db.isMaster()
{
	"hosts" : [
		"your_current_ip:27017",
		"your_current_ip:27018",
		"your_current_ip:27019"
	],
	"setName" : "rs",
	"setVersion" : 1,
	"ismaster" : true,
	"secondary" : false,
	"primary" : "mongo1:27017",
	"me" : "mongo1:27017",
    ...
}
```

Connect to Mongodb with replica set:

```
$ mongo "mongodb://localhost:27017,localhost:27018,localhost:27019/?replicaSet=rs"
```


### 3. Try failover

Let's create some data to db:

```
> use testdb
> db.createCollection('data')
> db.data.insert({name: "first doc"})
Inserted 1 record(s) in 6ms
WriteResult({
  "nInserted": 1
})
> db.data.find()
{
  "_id": ObjectId("5b2d0a6591c0126271b547ef"),
  "name": "first doc"
}
Fetched 1 record(s) in 6ms
```

Stop the primary one:

```
$ docker stop mongo1
mongo1
$ docker ps -a
CONTAINER ID        IMAGE                  COMMAND                  CREATED             STATUS                      PORTS                                            NAMES
377429727de4        mongo:latest           "docker-entrypoint.s…"   3 minutes ago      Exited (0) 6 seconds ago                                                     mongo1
986274ddfc0d        mongo:latest           "docker-entrypoint.s…"   3 minutes ago      Up 28 minutes               0.0.0.0:27019->27017/tcp                         mongo3
8dec66375f49        mongo:latest           "docker-entrypoint.s…"   3 minutes ago      Up 28 minutes               0.0.0.0:27018->27017/tcp                         mongo2
```

After few seconds, Replica set will elect a new primary as the previous time.

Check data still exists.

```
> db.data.find()
{
  "_id": ObjectId("5b2d0a6591c0126271b547ef"),
  "name": "first doc"
}
Fetched 1 record(s) in 2ms
```

Restart the mongo1:

```
$ docker start mongo1
```
