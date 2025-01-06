# System Design
This repo contains high level system design for few applications.

## Table of content
[Ride Booking App](#Ride-Booking-App)\
[Messaging App](#Messaging-App)\
[Food Delivery App](#Food-Delivery-App)

## Ride Booking App

This is a system design for a ride booking app like Uber, Lyft etc.

### Users

- User: Client who is trying to book the ride.
- Driver: Client who drives the vehicle.

### Functional Requirements

- User can see cab options with information like ETA, pricing etc.
- User can book a cab of its choosing.
- Driver can choose to accept or deny the ride.
- Driver can start and end the ride.
- Driver and Users can share their location.
- User can pay for the ride.
- User can rate the ride after its over.

### Non - functional Requirements

- Low latency
- Highly available
- Highly scable

### Estimations

Assume 100 million daily users and 10 million rides booked with 10 million drivers.

1. Number of requests
   Assume each user make 10 actions daily = 100 mil x 10
   = 10^9 requests per day
   = 10^9 / (24 x 60 x 60)
   = 12K requests/ sec

2. Bandwidth requirement
   Assume User and driver send location every 3 sec during rides each msg is 30 bytes.
   Assume each action made is also 100 bytes each.
   Total data = (100 bytes x 100 mil x 10) + ((10 mil x 30 x 24 x 3600) / 3) = 100 bil + 8640 bil = 9 TB
   Bandwidth = 9 TB / 24 x 3600 = 1 GB / 24 x 36 = 100 MB / sec

### APIs

1. Find cabs from source to destination.\
   `getRide(api_dev_key: string, userID: string, source: vector<double>, destination: vector<double>): vector<Cabs>`\
   This API takes source and destination of the User and return list of cab options, cab types with pricing and ETA.

2. Select cab\
   `bookRide(api_dev_key: string, userID: string, source: vector<double>, destination: vector<double>, cabType: ENUM<string>): Ride`\
   This API takes source, destination and cabType and return the ride.

3. Accept ride / Deny ride\
   `acceptRide(api_dev_key: string, rideID: string): boolean`\
   `denyRide(api_dev_key: string, rideID: string): boolean`\
   These APIs allows driver to accept or deny the ride.

4. Start ride / end ride\
   `startRide(api_dev_key: string, rideID: string): boolean`\
   `endRide(api_dev_key: string, rideID: string): boolean`\
   These APIs allows driver to start or end the ride.

5. Pay for the ride\
   `pay(api_dev_key: string, rideID: string, paymentID: string): boolean`\
   This API allows user to pay for the ride. Payment APIs needs to be **idempotent** so payment retries don't ruin consistency of data.\

6. Rate the ride\
   `rateRide(api_dev_key: string, rideID: string, start: int, feedback?: string): boolean`\
   This API allows user to rare the ride after completion.

### Data Models

1. User: userID, userName, location etc
2. Driver: driverID, dirverName, location etc
3. Ride: rideID, userID, driverID, source, destination etc
4. Payment: paymentID, rideID, userID, amount, status

### API Flow

1. User requests for ride options from Ride service.
2. Ride service sends cab options to user.
3. User selects an option and book a ride.
4. Ride service calls Proximity service to find drivers in radius.
5. Selected driver accepts the ride.
6. Ride service sends the ride to User and driver starts the ride.
7. Driver sends the location to driver service to save in database for tracking.
8. Once ride is over, User can pay and rate the ride.

### Services

1. User Service: Handles user details, authentication and authorization.
2. Driver Service: Handles driver details, sharing location of driver.

- The location shared by driver can be done by pull or push methods: either driver push the location to service or service pulls the location from driver.
- We can keep websocket connection with driver to send the location every few seconds.

3. Ride Service: Handles rides, shows ride options, start / end ride.
4. Proximity Service: Finds drivers in proximity.

- Main concern is to find the drivers in proximity fast and rank them in preference order.
- We can find the driver in radius by doing simple SQL query, like
  `SELECT * FROM driver WHERE lat BETWEEN lat - Radius AND lat + Radius AND WHERE long BETWEEN long - Radius AND long + Radius;`
  But this will be very slow since we need to traverse each row. So we need better way to store the location.
  We can store the location as Quad tree or GeoHash.
- We can use GeoHash, that allows us to store the location as a simple string of few chars based on precision we want, 6-7 chars should be enough to give us drivers in around 2km radius.
  We can keep the table indexed at location geohash, this will make finding driver very fast.
- Nearest drivers can lie on the edge of the current GeoHash cell, so neighboring cells should also be queried.

5. Payment Service: Completes payment for the ride.

- Payment service can have multiple payment options like credit card or paypal. We can have strategy design pattern.

6. Notifcation Service: Sends notification to client.

- We can have kafka queue, from where we can send notification to clients.

### Technological Decisions

1. Databases
   The data is mostly relational so we can use MySQL database.

2. Scalability

- With lots of requests to our system, we will have a lot of load on our services, we can use load balancer to send requests to one of the server.
- All databases can have read replicas and have leader-follower setup to update the replicas with new data.
- Data partioning / **sharding** can be done on driver database based on location. We can have consitent hashing to decide the database server to get the data from.
- We can also have **Rate limiting** to ensure the system is not overwhelmed.

3. Low latency

- We can have **Redis cache** at Ride service to store cab options and all for common locations for user. We can use **LRU** as cache invalidation technique.

## Messaging App

This is system design of a messaging app like Whatsapp, WeChat etc.

### Functional Requirements

- User can send messages to each other, messages can contain photo, video, gif etc.
- User can send messages to groups with upto 100 members.

### Non - Functional Requirements

- Highly available, consistency is not mandatory, since a delayed msg or lack of sync between devices is fine.
- Low latency
- Highly scalable

### Estimations

Assume daily active users are 100 million, each sends 20 msgs a day.

1. Number of requests
   100 mil x 20 = 2 bil requests
   Per sec, 2 bil / (24 x 60 x 60) = 24K requests / sec

2. Storage
   Total messages sent = 100 mil x 20 = 2 billion msgs

Assume 5% messages contain media, total messages = 0.05 x 100 mil x 20 = 100 million msgs
Say, each msg is 200 bytes in size and media size in 100KB.
Total data = (2 bil x 200) + (100 mil x 100KB) = 4 x 10^11 + 10^13 ~ 10 TB

3. Bandwidth
   Total data per day = 10TB, per sec, 10 TB/ (24 x 60 x 60) ~ 120MB / sec

### Data Models

1. User: This will keep user details like name, contact details etc.
2. Chat: chatID, senderID, recieverID.
3. Group: groupID, groupName etc.
4. ChatMessage: msgID, channelID (chat or group), content, mediaURL etc.
5. GroupMember: userID, groupID.

### APIs

1. Send message\
   `sendMessage(api_dev_key, userID: string, content: string, mediaURL: string): boolean`

2. Join / Leave group\
   `joinGroup(api_dev_key, groupID, userID): boolean`\
   `leaveGroup(api_dev_key, groupID, userID): boolean`

3. Get all messages\
   `getMessages(api_dev_key, userID: string, channelID): vector<ChatMessage>`

4. Get all chats / groups\
   `getChats(api_dev_key, userID: string): vector<Chat> | vector<Group>`

### Services

1. User Service: This service deals with user details and operations like authentication and authorisation.
2. Chat Service: This Service deals with getting/ sending messages between clients.
3. Group Service: This service deals with group member addition, removal etc.
4. Notification Service: This service deals with sending notification to user when offline.
5. Presence Service: This service deals with keeping record of when was user last active.

## Technological Decisions

1. Chatting
   We can have users getting messages via pull or push methods. Users can either pull messages from database or our server can push new messages to the user.
   **Pull method:** User does long polling and keep asking Chat service for new messages, this will put extra strain on service, since most of the time response will be empty.
   **Push method:** Service pushes new messages to client when new messages arrives, so we will go with push method.

2. Connection
   We can have **websocket** connection with client for sending and receiving messages. This allows bidirectional sharing of messages from both ends.

3. Database
   The structure of data is fixed, so relational database works well for us. We can use MySQL or Postgres. We can also have multiple **read replicas** and **leader-follower** policy to update read replicas.
   We can do further **sharding** and partioning for faster message fetching, and do consistent hashing to choose database server.

4. Caching
   We can do caching for userID, to fetch chats and messages. We can use LRU for cache invalidation. We can use **Redis cache**.

5. Last seen
   To keep track of last activity of user, we can either have a **heartbeat** to check the connection OR we can keep track of last acitivty of user to decide its active or not. If not active, we can put message in **Kafka** queue that would take the message to notification service that would send the notification to the client using serivces like **Firebase Cloud Messaging**.

6. Media storage
   Media can be stored in **blob storage like S3** and can be served to users using **CDN**.

## Food Delivery App

This is system design for a food delivery app like Swiggy, Zomato etc.

### Users

User: client ordering food.
Restaurant: restaurants with menu and food items.

### Functional Requirements

- User can find restaurants in the area.
- User can view menu of restaurants.
- User can order food from restaurant.
- Retaurant can add menu items.
- Delivery agent can choose to accept / deny an order.
- User can track the order.

### Non - Functional Requirements

- High availability
- Low latency
- Highly scalable

### Estimations

Assume we have 5 million daily active users. And each user perform 10 actions each day. And 100K orders are made daily.

1. Number of requests
   Total requests = 5 mil \* 10 = 50 mil requests / day
   = 50 mil / (24 x 3600)
   = 600 requests / sec

2. Data Storage
   Assume each action message is about 400 bytes in size.
   Total size = 600 x 400 = 240000 bytes = 0.24MB

Assume delivery agent location is shared every 5 sec for each order,
Total size = 100 x 10^6 x (24 x 3600 / 5) = 10^8 x 18000 = 2 TB

Also to store menu and restaurant, say about we have 100K restuarants and each menu is about 2KB,
Total size = 100 x 10^3 x 2 x 10^3 = 200MB

Total storage needed ~ 2 TB
Total storage for 10 years = 2TB x 365 x 10 ~ 8 PB

3. Bandwidth requirement
   Total data / day = 2 TB / (24 x 3600) = 23 MB / sec

### API flow

1. User search nearby restaurants.
2. User selects restaurants and view their menu.
3. User orders from food menu.
4. Delivery agent accepts the order and delivers the order.
5. Delivery agent share location so user can track.

### Data Models

1. User: This class keeps track of user information, userID, userName, userLocation etc.
2. DeliveryAgent: This class keeps track of delivery agent information, agentID, agentName, agentLocation etc.
3. Restaurant: This class keeps track of restaurant information.
4. MenuItem: This class keeps track of individual menu item with price, name, menuID etc.
5. Menu: This class will keep menuID, restaurantID, tracking which menu belongs to which restaurant.
6. Cart: This class keeps userID, cartID.
7. CartItem: This class keeps cartID, menuItemID.
8. Order: This class keeps orderID, userID, userLocation, paymentID, status.
9. Payment: This class keeps paymentID, orderID, userID, status.

### APIs

1. View restaurants, `/restaurant`\
   `getRestaurants(api_dev_key, userID: string, userLocation: vector<double>): vector<Restaurant>`

2. View menu of a restaurant, `/menu/{id}`\
   `getMenu(api_dev_key, restaurantID: string): Menu`

3. Add item to cart, `/cart/{id}`\
   `addItem(api_dev_key, menuID: string, userID: string, menuItemID: string): boolean`

4. Place order\
   `placeOrder(api_dev_key, userID: string, cartID: string, userLocation: vector<double>): orderID: string`

5. Accept / Deny order\
   `acceptOrder(api_dev_key, orderID): boolean`\
   `denyOrder(api_dev_key, orderID): boolean`

6. Start / End order\
   `startOrder(api_dev_key, orderID): boolean`\
   `endOrder(api_dev_key, orderID): boolean`
7. Share location\
   `shareLocation(api_dev_key, location: vector<double>, agentID: string): boolean`

8. Pay for the order\
   `pay(api_dev_key, orderID: string, userID: string, paymentID: string, paymentMethod: ENUM<string>): boolean`

9. Add menu items\
   `addMenuItem(api_dev_key, restaurantID: string, menuID: string, item: string): boolean`

10. Add menu\
    `addMenu(api_dev_key, restaurantID: string, menuID: string): boolean`

### Services

1. UserService
   This service handles user information CRUD, authentication and authorisation.

2. DeliveryAgentService
   This service handles delivery agent information CRUD.

3. ProximityService
   This service finds restaurants and delivery agent based on user and location proximity.

4. OrderService
   This service handles placing order with user cart.

5. PaymentService
   This service handles payment for the order.

6. LocationService
   This service handles keep track of delivery agent location so the user can track the order while its in transit.

7. RestaurantService
   This service handles restaurant information CRUD and adding of menu items.

8. NotificationService
   This service handles sending notification to user. Each service produces message for the queue when order status changes, which is then notified to user.

### Technological Decisions

1. Database selection
   The structure of data is fixed, so relational database works well for us. We can use MySQL or Postgres. We can also have multiple **read replicas** and **leader-follower** policy to update read replicas.
   We can do further **sharding** and partioning for faster data fetching, and do consistent hashing to choose database server.

2. Finding restaurants and delivery agents

- We need to find restaurants and delivery agents nearby fast and efficiently and rank them in preference order.
- We can find the agent or restaurant in radius by doing simple SQL query, like
  `SELECT * FROM agent / restaurant WHERE lat BETWEEN lat - Radius AND lat + Radius AND WHERE long BETWEEN long - Radius AND long + Radius;`
  But this will be very slow since we need to traverse each row. So we need better way to store the location.
  We can store the location as Quad tree or GeoHash.
- We can use GeoHash, that allows us to store the location as a simple string of few chars based on precision we want, 5 chars should be enough to give us drivers in around 10km radius.
- We can keep the table indexed at location geohash, this will make finding agent or restaurant very fast.
- Nearest delivery agent or restaurant can lie on the edge of the current GeoHash cell, so neighboring cells should also be queried.

3. Location tracking
   When delivery agent is in transit, we need to track the agent. The location sharing can be websocket connection otherwise each request will take a lot of time.

4. Payment

- At low level we can have multiple payment algorithms, chosen using strategy design pattern.
- Introduce a retry mechanism for payments in case of transient failures. Ensure idempotency to handle duplicate payment requests safely.
- Use distributed transactions or write-ahead logging to avoid partial failures.

5. Notifications
   We can kafka queue setup, which takes order status from services and sends notification to user, using external services like Firebase Cloud Messaging.

6. Order Management
   when order status changes, message is produced to a kafka topic, from which the next service involved picks up message, performs its operation and produces message for next service.
   Eg, order service will produce msg for delivery agent service, which finds driver using proximity service, then assign agent and produce msg to location service, which picks it up and starts the tracking of order.

7. Caching

- Popular restaurant data and menus can be cached for the user. We can use simple Redis cache setup for this.
