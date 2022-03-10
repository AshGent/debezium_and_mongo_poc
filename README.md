

In this POC we will be utilizing X terminal windows to monitor the different pieces and work with the containers. A lot of this can be done in one session, however the influx of logs can be overwhelming and hard to trouble shoot.

All terminal sessions should be at the root of the repo

---
# Start most of the containers
## Terminal 1
```
$ docker-compose up zookeeper
```

## Terminal 2
```
$ docker-compose up kafka
```

## Terminal 3
```
$ docker-compose up mongodb
# There will be an influx of logs with the message "Failed to refresh key cache" - this will be addressed shortly. Ignore for now
```
---

# Setup Mongodb Replication set and create dummy data
## Terminal 4
```sh
# This will open an interactive session into the running mongo container
$ docker container exec -it <mongo_container> /bin/bash

# Run these mongo commands
# initiate replica set
root@mongodb: mongo localhost:27017/inventory <<-EOF
    rs.initiate({
            _id: 'rs0',
            members: [
                {
                    _id: 0,
                    host: 'mongodb:27017'}]});
EOF

# Create admin user
mongo localhost:27017/admin <<-EOF
    db.createUser({
        user: 'admin',
        pwd: 'admin',
        roles: [ { role: "userAdminAnyDatabase", db: "admin" } ] });
EOF

# Create roles to create roles 'listDatabase' & 'readChangeStream'
mongo -u admin -p admin localhost:27017/admin <<-EOF
    db.runCommand({
        createRole: "listDatabases",
        privileges: [
            { resource: { cluster : true }, actions: ["listDatabases"]}
        ],
        roles: []
    });
    db.runCommand({
        createRole: "readChangeStream",
        privileges: [
            { resource: { db: "", collection: ""}, actions: [ "find", "changeStream" ] }
        ],
        roles: []
    });
EOF

# Create a debezium user
mongo -u admin -p admin localhost:27017/admin <<-EOF
    db.createUser({
        user: 'debezium',
        pwd: 'dbz',
        roles: [
            { role: "readWrite", db: "inventory" },
            { role: "read", db: "local" },
            { role: "listDatabases", db: "admin" },
            { role: "readChangeStream", db: "admin" },
            { role: "read", db: "config" },
            { role: "read", db: "admin" }
        ]
    });
EOF

# Create some dummy data
mongo -u debezium -p dbz --authenticationDatabase admin localhost:27017/inventory <<-EOF
    use inventory;
    db.products.insert([
        { _id : NumberLong("101"), name : 'scooter', description: 'Small 2-wheel scooter', weight : 3.14, quantity : NumberInt("3") },
        { _id : NumberLong("102"), name : 'car battery', description: '12V car battery', weight : 8.1, quantity : NumberInt("8") },
        { _id : NumberLong("103"), name : '12-pack drill bits', description: '12-pack of drill bits with sizes ranging from #40 to #3', weight : 0.8, quantity : NumberInt("18") },
        { _id : NumberLong("104"), name : 'hammer', description: "12oz carpenter's hammer", weight : 0.75, quantity : NumberInt("4") },
        { _id : NumberLong("105"), name : 'hammer', description: "14oz carpenter's hammer", weight : 0.875, quantity : NumberInt("5") },
        { _id : NumberLong("106"), name : 'hammer', description: "16oz carpenter's hammer", weight : 1.0, quantity : NumberInt("0") },
        { _id : NumberLong("107"), name : 'rocks', description: 'box of assorted rocks', weight : 5.3, quantity : NumberInt("44") },
        { _id : NumberLong("108"), name : 'jacket', description: 'water resistent black wind breaker', weight : 0.1, quantity : NumberInt("2") },
        { _id : NumberLong("109"), name : 'spare tire', description: '24 inch spare tire', weight : 22.2, quantity : NumberInt("5") }
    ]);
    db.customers.insert([
        { _id : NumberLong("1001"), first_name : 'Sally', last_name : 'Thomas', email : 'sally.thomas@acme.com' },
        { _id : NumberLong("1002"), first_name : 'George', last_name : 'Bailey', email : 'gbailey@foobar.com' },
        { _id : NumberLong("1003"), first_name : 'Edward', last_name : 'Walker', email : 'ed@walker.com' },
        { _id : NumberLong("1004"), first_name : 'Anne', last_name : 'Kretchmar', email : 'annek@noanswer.org' }
    ]);
    db.orders.insert([
        { _id : NumberLong("10001"), order_date : new ISODate("2016-01-16T00:00:00Z"), purchaser_id : NumberLong("1001"), quantity : NumberInt("1"), product_id : NumberLong("102") },
        { _id : NumberLong("10002"), order_date : new ISODate("2016-01-17T00:00:00Z"), purchaser_id : NumberLong("1002"), quantity : NumberInt("2"), product_id : NumberLong("105") },
        { _id : NumberLong("10003"), order_date : new ISODate("2016-02-19T00:00:00Z"), purchaser_id : NumberLong("1002"), quantity : NumberInt("2"), product_id : NumberLong("106") },
        { _id : NumberLong("10004"), order_date : new ISODate("2016-02-21T00:00:00Z"), purchaser_id : NumberLong("1003"), quantity : NumberInt("1"), product_id : NumberLong("107") }
    ]);
EOF
```
---

# Starting Kafka Connect
## Terminal 5
```sh
$ docker-compose up connect
```

## Terminal 6
```sh
# Check status of kafka connect
$ curl -H "Accept:application/json" localhost:8083/
# should return something like: {"version":"3.0.0","commit":"cb8625948210849f"}

# Check existing connectors (should be none)
curl -H "Accept:application/json" localhost:8083/connectors/

# Create a new connector
curl -i -X POST -H "Accept:application/json" -H "Content-Type:application/json" localhost:8083/connectors/ -d @inventory-connector.json

# If you need to delete the connector for some reason
curl -X DELETE localhost:8083/connectors/inventory-connector


# Get your docker assigned network name
# This will be used to connect to the network to monitor the kafka topics
docker network ls
# ex: single_replica_default
```
---

## Terminal 7
```sh
# In a new terminal
# This open a terminal in a new kafka container that is linked to kafka and zookeeper
docker container run --rm -it --link <zookeeper_container>:zookeeper --link <kafka_container>:kafka --net <docker_network> debezium/kafka:1.8 /bin/bash

# In the new docker terminal
# We'll start monitoring the customers collection in the database
$ [root@host] /kafka/bin/kafka-console-consumer.sh --bootstrap-server=kafka:9092 --from-beginning --topic dbserver1.inventory.customers

```

## Terminal 4
### Using the terminal that already has an interactive session open
```sh
docker container exec -it --rm <mongo_container> /bin/bash

$ [root@host] mongo -u debezium -p dbz -authenticationDatabase admin localhost:27017/inventory;
$ [root@host] use inventory;
$ [root@host] db.customers.insertOne({first_name: 'A Name', last_name: 'B Name', email: 'C Email'});
```

If you return to terminal 7 you should now see a record appear that contains a before and after state of the record you inserted.
