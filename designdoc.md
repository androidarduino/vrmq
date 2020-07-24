# Entanglement: A MQ Supports Syncing #

## Senarios ##

This is a middleware that provides applications(desktop, mobile app, webpages, etc.) the ability to communicate to each other, and sync data to each other.

Although you can send raw messages to your client groups to your private channels, we recommend you use it in a sync fashion, which automatically updates any changes to all the subscribed parties in the topic.

## API Specification ##

1. Connection and Authentication: connect to the HUB server, authenticate self with a unique user id and permissions. How the user authenticate the client is to be determined.
2. Subscription and Unsubscription: each user connected to the HUB could subscribe to a topic, as listener, speaker, or both, with specific update preferences and frequencies. Of course unsubscription facilities would be provided as well. After the last participants unsubscribes from a topic, the HUB will terminate that topic and dump all the transactions into cold storage for future reference.
3. Updating and AutoSyncing: user with speaker permissions can update the topic data, all the subscribers will get this resource update. An application can have multiple syncing topics. Each topic represents a set of specific serialized resources, eg. the value of a text field on the interface, or the combined data of a set of variables in the cross platform application. The resources are hashed to produce a unique identification string representing the topic.
4. Regarding data type for syncing, since we are making a cross-platform, Internet framework that works with multiple languages, minimizing the data type supported is essential. In the first version we plan to support the following basic types only: **integer**, **string**, **float,** and collection types of the basic types: **array**, **map**. If you want to support your own types, serialization and deserialization functions must be brought by yourself.



```go
//Golang API Examples:

//connect to the HUB, eg localhost, 8090, "www.vrcats.com/werewolf"
func ConnectToHub(serverIp int, port int, applicationSpace string)

//subscribe to a topic, eg. "screen.playerStatus", or "screen.currentPlayingBGM"
func SubscribeTo(topic string)

//update a topic with syncing data
func Update(topic string, data string, version int)
```



```javascript
//Javascript side API Examples:

function connectToHub(serverIp, port, applicationSpace)

function subscribeTo(topic)

function update(topic, data, version)
```



## HUB Design ##

### Business Logics ###

* On connection
  * Create or load user profile, maintain websocket connection to the user
  * Keep the user the list of the application
  * Each client should have a persistent UUID attached to them which never changes
* On first message to a topic
  * Verify the topic is valid for the application
  * Verify the message is valid in size and format
  * Create and add the topic to the topic list
  * Subscribe the sender to the topic
  * Multicast the message to the subscribers of the topic that matches application
* On subscription of a topic
  * Add a topic to (user+application) mapping to the subscription list if it is a subscription
  * Force a update of latest version of the topic data cached in the server
* On unsubscription of a topic
  * Remove a topic to (user+application) mapping from the subscription list if exists
  * Check for topics, if no subscription left, remove it from the topic list
* On connection closed
  * Add connection user to the scheduled janitor list, if it is not reconnected in a certain period, say 60 seconds, remove its profile from the connection list
  * Check for topics and related subscriptions, if they are orphaned (no more user is subscribed to the topic), remove them immediately from the list
  * If application is orphaned, dump all application context, subscriptions into persistent storage, remove the application instance, dump all related information into logs
* On update message
  * Verify the topic, user, applications are valid
  * Verify the message is valid in size and format
  * If it is a diff message, try apply it to the latest version, ensure it fits, otherwise reject the message
  * Maintain the latest version in persistent storage
  * Push it to the topic history to persistent storage, create snapshots if available
  * Broadcast the message to all subscribers
* On retrieving version
  * Veirfy the user is valid to retrieve the verison for the topic and application
  * Query the storage for latest version and send it back to the requester
* On reconnect
  * Load the application context from persistence storage
  * Rebuild the subscriptions, topics, versions of the application
* On scheduled task
  * Find related subscriptions of the topic and application
  * Send updates to all subscribers
  * And example of the scheduled task is heartbeat

### Backend Storage ###

* Key-value store to store:
  * Application context dumps
  * Subscriptions
  * Users and application registrations
  * Credentials and related information
* Log stream to store:
  * Activities

## Data Format and Protocols ##

1. Command protocol: JSON with compulsory verb and parameters, followed with me
2. Message format:
   1. Size: < 256k each message
   2. Size * Subscribers < 1M for each update
   3. Diff update is preferred all the time

## Workflow Example ##

```sequence
Writer->HUB: connect user=12345
Note left of HUB: verify credentials of Client1
HUB->Writer: ok connected 12345
Reader->HUB: connect user=45678
Note right of HUB: verify credentials of Client2
HUB->Reader: ok connected 45678

Writer->HUB: update topic=vrcats/werewolf msg=led1:0f0c0
Note left of HUB: verify application 002
Note left of HUB: create topic werewolf 0034
Note left of HUB: add subscription user=12345 topic=0034 access=rw
Note left of HUB: multicast topic=0034 msg=led1:0f0c00

HUB->Writer: topic=0034 msg=led1:0f0c00

Reader->HUB: subscribe topic=vrcats/werewolf access=ro
Note right of HUB: verify credentials
Note right of HUB: add subscription user=45678 topic=0034 access=ro

Reader->HUB: update topic=vrcats/werewolf msg=led1:000000
Note left of HUB: verify message
Note left of HUB: multicast topic=0034 msg=led1:000000

HUB->Writer: topic=0034 msg=led1:0f0c00
HUB->Reader: topic=0034 msg=led1:0f0c00

```



## SDK Design ##

We are using Javascript as an example. The same logic applies to other programming language SDKs. Some syntax candy could be provided in the SDK. 

**Syncing logic** is the easiest way to use this framework. For example the user could use the framework like:

```javascript
//The following command will bind the variables in the array to a remote resource for syncing. Anytime there is an update for any of these variables, the Javascript SDK will automatically get notified and update the visible elements in the DOM. Developers don't need to worry about protocols and time sequences, reconnection, etc. at all. Simply bind your local variables to remote resources and watch them get updated in real time.

var subscription = entanglement.subscribeListenAndCast(
  "123.34.5.6:8990/www.vrcats.com/werewolf/screen.playerStatus",
  [
    Document.title, LED.style.color,
    Player1_Name.text, Player1_img.src, Player1_bg.src,
    Player2_Name.text, Player2_img.src, Player2_bg.src,
    Player3_Name.text, Player3_img.src, Player3_bg.src,
    Player4_Name.text, Player4_img.src, Player4_bg.src,
    Player5_Name.text, Player5_img.src, Player5_bg.src,
    Player6_Name.text, Player6_img.src, Player6_bg.src
  ]
);

//On the other hand, if there is changed locally, simply call this API to sync the change to all other people who subscribed to this topic:

subscription.broadcast();

//To force an update, simply call:

subscription.getUpdate();
```

```javascript
//TODO: need to carefully consider racing conditions of updates and the best approach to minimize the impact to SDK and users
//Yet another example of transaction notification, where eventQueue is a fifo queue records all the user button pressing events.

var subscription = entanglement.subscribeCast(
	"123.34.5.6:8990/www.vrcats.com/werewolf/event.queue",
  eventQueue
)

//User can safely update the local variable using provided function:
entanglement.safeUpdate(eventQueue, newEventQueue);
//Then remote sides will updated as well.

//If there are more than one caster updating the same fields at the same time, multiple user update conflicts must be considered carefully.
```

You can also use the framework as a simple **message queue**, the message will be forwarded to the destination by the HUB, however on the receiptant side the message must be correctly handled.

```javascript
//Send a message to a specific user in the same application
entanglement.sendMQ(
	message, receiverId
)

//Send a message to all subscribers in a topic
subscription.sendRawMessage(
	message
)
```



## Scalability Considerations ##

* HUB should be horizontally able to scale out
* Applications could be evenly spreaded into HUB farms if available using hash
* An application could reside on a subset of the HUB far if needed. A load balancer will distribute the connections evenly in the host list
* If one topic has extrem high subscribers, listening subscribers should be load balanced into multiple hosts. The hosts should be able to talk to each other on updates and multicasts. However all casting subscribers should be on the same host to maintain data consistency
* On host failures, the load balancer host should connect the user to the same HUB farm list when user tries to connect again
* Auto scaling of the HUB farm should be considered

## Security Design ##

TBD

## Deployment Design ##

* HUB cluster needs to be deployed before user can start to use
* Credentials and authorizations of applications needs to be created

## Operation Considerations ##

* HUBs needs to be janitored regularly

* User should be able to update credentials and authorizations on the fly

  

## Modes Of MQ ##

There are two major use cases in MQ:

* Stream: append message to subscribers each time
* Update: update an existing object each time, could be the whole object as a message, or delta update

There are four different mapping modes for MQ:

* 1:1 One message source, goes to one destination
* 1:n One message source, goes to many destinations
* n:1 Many message sources, goes to one destination
* m:n Many message sources, goes to many destinations

Different strategies should be used to best handle different use cases and mapping modes.



## New Design ##

1. For Stream based solution, design a two way stream object in SDK, the user can send/receive messages using the stream object
2. For Update based solution, design a binding of objects machenism in SDK, allow user to bind any variable to our updater, who can handle the update automatically
3. For languages that support messaging like Golang, Qt, Erlang, extend the native supports to make it easier to learn and use



### Examples ###

1. Stream based:

Stream.objectA << "test message";

Stream.objectA.onMessage = callBack(...);

2. Update based:

Updater.sync(objectA);

Updater.objectA.onMessage = callBack(...);

### Detailed Design for Client Side ###

1. WSDir: An object that can look up the directory and send messages to the correct hub
2. Stream: An object for Stream data communication
3. Updater: An object for Update syncing with delta data support
4. Subscriber: A manager for subscribing/unsubscribing stuff
5. WSConnMgr: A connection manager to maintain websocket connections

### Detailed Design for Server Side ###

1. A http websocket server to maintain connections to all clients
2. A client directory service center
3. A subscription service center
4. A traffic control/failsafe manager for the service

5. An anti attack service to protect server

6. Each of the domain will reside on at least three replicas in different DCs.
7. The host will be able to split/merge according to traffic changes

