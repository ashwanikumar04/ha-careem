#Design Document

![Design Diagram](https://github.com/ashwanikumar04/ha-careem/blob/master/Careem.svg)
![Design Diagram](https://github.com/ashwanikumar04/ha-careem/blob/master/Careem.png?raw=true)


#Databases Used:

1) Postgres for Storing Customer, Driver and Trip Information

2) Distributed REDIS(geocoding): To store driver location and retrieve nearest drivers based on customer location

3) Couchbase to store location update logs and Trip logs.

4)REDIS/MEMCACHE: to be used by trip server for in-memory storage.

#Messaging Queues(pub-sub)

1) ACTIVEMQ cluster as publish subscribe system to route trip requests to driver and
update trip information to customer at regular interval


#AWS Infra:

1) EC2 Instances for Trip Server(with AutoScaling), Location Service, hosting Message Queues, Couchbase DB

2) RDS Cluster for Postgres DB

3) AWS Elastic Cache: REDIS/MEMCACHE

4) Elastic LoadBalancer for Trip Service Instances with request routing
configured based on Trip ID(has prepended Server ID)

5) Elastic LoadBalancer for Location Service Instances

Message Exchange can be done in JSON, MsgPack or ProtoBuf.

We can choose ProtoBuf because it's efficient on little data and can save a lot of money on network calls.

Design Explanation:

Initial Setup:

Once Driver logins to the system, a Virtual Topic with Driver ID gets created on ACTIVEMQ Cluster,
the mobile app subscribes to the created topic via MQTT.

Customer logins to the system, a Virtual Topic with Customer ID gets created on  ACTIVEMQ Cluster

Driver location is pushed to Location Service after every 10 seconds(configured interval) from his mobile app.

Location Server updates the location data in REDIS and logs it on Couchbase DB.

#Customer App makes request for Trip:

1) The request gets routed to a Trip Server Instance via Elastic Load Balancer with his location coordinates as parameter

2) The Trip Server Instance makes a query to Query Service and retrieves the nearest N drivers in the preconfigured range.

3) The Trip Server then creates a trip object with Trip ID as TripServerID+GUID and
publishes a message with the Trip ID along with the list of driver Ids as filter to ACTIVEMQ.

As such only specific drivers only receive the message.

4) The Trip Object is saved in local REDIS cache as well on Postgres DB.

4) One or more driver may accept the trip request.

5) The request accept messages along with the Trip ID is published to ACTIVE MQ by extracting Server ID from the TRIP ID and key will be ServerID+Accepted.

6) Trip Service instance has a listener(subscriber) for TripServerID+Accepted as the topic. One listener for each Server ID.

7) The trip service puts all the acceptance messages in a local Messaging Queue(RabbitMQ).

8) One listener processes one message at a time and assigns the first Driver to the Trip.

9) Server pushes a message to ACTIVE MQ with the Trip data and Customer ID as key as well as Driver ID as key.

This enables the Customer and Driver to receive the trip information.

Similarly All events like trip start, trip location updates, trip end and billing gets routed via ACTIVE MQ cluster.

