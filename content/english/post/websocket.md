+++
author = "Julien Vermillard"
title = "Adding WebSocket to a REST backend"
tags = [ "backend", "websocket", "redis", "pubsub"]
date = "2023-12-12"
+++

As a software architect, I often need to deal with backend systems that provide a variety of REST APIs, each supporting a set of CRUD (Create, Read, Update, Delete) operations. Traditionally, these systems haven’t offered a way for clients to subscribe to real-time notifications. So, clients are forced to either continuously poll the server or require users to manually refresh their browsers, which can be a frustrating user experience in 2023.

Fortunately,WebSocket is now a reliable standard, widely implemented across major web browsers. Additionally, most advanced enterprise proxies and load balancers now support WebSocket, rendering older methods like long polling and server-sent events largely obsolete. Though, from a personal perspective, I still find server-sent events to be an elegant solution, especially within the context of HTTP protocols, due to their simplicity and efficient unidirectional data flow.

How can we efficiently deliver event notifications to the UI client? A straightforward approach involves integrating event handling at specific points in our server-side code. This can be done either when generating the JSON response for an API call or at the DAO (Data Access Object) level, depending on what best aligns with your architecture. The key is to create a JSON document that captures the essence of the event. This document should include the type of event (create, update, or delete), along with essential details like:

* The name of the entity
* Its unique identifier
* The content of the entity (when applicable)

In the case of a deletion, including the entity’s content is optional. For updates, the approach can vary based on the specific needs of your application. You might choose to provide only the new version of the content, both the old and new versions, or none at all. In scenarios where the content is not provided, the user can poll the API to retrieve the updated content if necessary. Personally, I often favor the latter approach. It helps avoid race conditions and the complexities inherent in distributed computing. Having a single channel for data retrieval can greatly simplify the system’s architecture.

![propagation of CRUD events from one user to another](/images/ws-event.png)

Now that we’ve outlined our data model, let’s delve into the scalability and design considerations of the backend to support it. A primary concern is to minimize traffic over the WebSocket channel between the browser and the server. To achieve this, it’s crucial to ensure that we only publish events that are relevant to a user’s current context within the application.

The strategy here involves allowing the UI to subscribe only to entities that are pertinent to the screen or page the user is currently viewing. My usual approach is to enable the client to define a list of subscription paths when initiating the WebSocket connection. If the user navigates to a different page, the application should close the current WebSocket connection and open a new one with updated subscriptions. The complexity of the application will dictate the nature of these subscriptions. For simpler cases, they might be a list of specific entities (like ‘user’, ‘group’, etc.). In more complex scenarios, where numerous entities are involved, grouping them into categories such as ‘rights-management’ (which might encompass ‘user’, ‘group’, ‘permission’ entities) can be more effective.

Being mindful of traffic is crucial when dealing with real-time updates. While loading a large JSON file during page initialization is manageable, continuously streaming numerous JSON messages can significantly slow down the web user interface. This is particularly true for users with high-latency or low-throughput wireless connections.

My advice is to start with a focused approach. Initially, target only those entities that are most critical to be updated in real time for your application. We are not aiming for a generic subscription mechanism for every entity in a REST API. Additionally, if your interest lies in backend-to-backend solutions, I would generally advise against using WebSockets, but that’s a topic for another discussion.

So, how do you implement a subscribe-publish feature atop an existing REST frontend and database backend? The process is relatively straightforward if you already have access to lightweight publish-subscribe systems like Redis or PostgreSQL. Both Redis Pub/Sub and PostgreSQL’s publish/subscribe capabilities are well-suited for this task. If you don’t have access to these, setting up a Redis instance solely for this purpose is a cost-effective and simpler alternative than using more complex solutions like RabbitMQ or Apache Kafka.

To bring this to life, you’ll need to adjust your backend application to publish these events. This involves modifying the code responsible for creating, updating, and deleting entities. Each event should be serialized and published to Redis, using a structured topic key to facilitate efficient distribution.

The topic key can be designed in various ways to optimize subscriber access. For example in Go pseudo-code you would need to:

```go
func Publish(topic string, tenantId string, entity string, eventType EventType, value interface{}) {
 msg := struct {
  Entity string           `json:"entity"`
  Type   EventType        `json:"type"`
  Value  interface{}      `json:"value"`
 }{
  Entity: entity,
  Type:   eventType,
  Value:  value,
 }

 bytes, err := json.Marshal(&msg)
 if err != nil {
   // log error
 } else {
     redis.Publish(tenantId + ":" + topic, bytes)
 }
}
```

In this example I added a tenant identifier to be able to segregate you data per customer/company for a multi-tenant SaaS like application.

Now on the subscription side, you need to add a websocket endpoint, for the UI to be able to subscribe to a given list of topics.

Again in pseudo Go code it would looks like:

```go
func ServeHTTP(w http.ResponseWriter, r *http.Request) {
  // first authenticate your user as for an API call
  ...

  // get the list of topic from the "topics" query argument
  topics := strings.Split(r.URL.Query().Get("topics"), ",")

  // for each topic subscribe
  for _, topic := range topics {
    tokens := strings.Split(topic, ":")

    if len(tokens) < 2 {
      w.WriteHeader(http.StatusBadRequest)
      w.Write([]byte("invalid topic: " + topic))
      return
    }
    tenantId := tokens[0]
    topicName := tokens[1]
  
    // Check the permission of the user, if you have such a model.
    ...
    
    // accept to upgrade HTTP to the websocket protocol
    c, err := websocket.Accept(w, r)
    if err != nil {
      log.Error("Websocket accept error: %v", err)
      if c != nil {
         c.Close(websocket.StatusInternalError, err.Error())
         return
      }
    }
    
    // subscribe to redis and publish the received events
    pubsub := redis.Subscribe(ctx, topics...)
    defer pubsub.Close()
    for {
      msg, err := pubsub.ReceiveMessage(ctx)
      if err != nil {
         panic(err)
      }
      payloadMsg := make(map[string]interface{})
      err = json.Unmarshal([]byte(msg.Payload), &payloadMsg)
      if err != nil {
         panic(err)
      }

       wsmsg := make(map[string]interface{})
       wsmsg["topic"] = msg.Channel
       wsmsg["entity"] = payloadMsg["entity"]
       wsmsg["type"] = payloadMsg["type"]
       wsmsg["value"] = payloadMsg["value"]

       data, err := json.Marshal(wsmsg)
       if err != nil {
         panic(err)
       }
       c.Write(ctx, websocket.MessageText, data)
    }
}
```

We end up with a solution looking like:

![](/images/ws-pubsub.png)

Happy to finally getting this post out of my writing queue. I hope it helps shed some light on a simpler, more efficient approach to real-time updates. With tools like Redis or PostgreSQL and a small amount of code we can say good bye to the relentless ‘refresh’ button. Who in 2023 has time to keep hitting refresh anymore :) ?
