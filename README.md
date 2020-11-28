## Prerequisites:
* Anypoint Platform Account (Deploy Mule Applications)
* Apache Kafka Instance (Used Confluent Cloud for Kafka Cluster)
* Slack Account (This is the client for our application)
* The Guardian Open Platform Account (To get News Content Via API)
* Alpha Vantage API (Get Real time Stocks via API)

## Idea Phase:
My Idea is to have a model that enable users (in our context *slack* users), to subscribe their own daily news snippets, stock information, such as latest quote for the stock etc., right from a single app, which in this case will be Slack.

### Thought Process
My first thought was to directly integrate the APIs and have the data stored in database and poll it every 3hr so that the news stays updated. Database poll will be heavy for I/O operation. I had done some R&D, and settle with Apache Kafka, since it is Low Latency in I/O operations, High Throughput and we can use it to design an architecture for our application to make it real time.

## Architecture:
![Real-time News Subscriber](https://dev-to-uploads.s3.amazonaws.com/i/fv4rr7kc9ca8ms6icxv1.png)

![Real-time News Publisher](https://dev-to-uploads.s3.amazonaws.com/i/35elqz419r7vjvx83tsl.png)

## Configurations
> Note: If you want to use the configuration I did, please skip till **Slack Configuration**

### Confluent Cloud Kafka Cluster
* Login/Register a free account in Confluent Cloud [here](https://confluent.cloud/signup).
* Create a Kafka Cluster. 
* Create two topics: `newstock-subscribe` and `newstock-unsubscribe`
* Create Kafka cluster keys to connect to kafka via MuleSoft Kafka Connector
* Go to the **Tools and Client Config** section of cluster page, there open 88clients** tab. Here you can see various language snippets for necessary credentials. Make not of them.

### Guardian API
* Visit this [page](https://open-platform.theguardian.com/access/).
* Register for Developer key. It will be sent to your registered email.

### Alpha Vantage API
* Visit this [page](https://www.alphavantage.co/support/#api-key) and enter the requested info. Key will be displayed once you submitted the form.

### Slack Configuration (*Required*)
* Create a Slack Workspace and register if you have not
* Visit (here)[https://api.slack.com/apps] to create an app
* Once App is created, click on *Slash Commands*. Click on Create Command as Shown below
![Slash Command Page](https://dev-to-uploads.s3.amazonaws.com/i/esp3xiknxrgzrsvm2f58.png)
* The below popup will be opened. Here you need to provide the following
![Slash Command Popup](https://dev-to-uploads.s3.amazonaws.com/i/y1hj96pz2nut52nh5w9c.png)
  * Slash Command: /subscribe 
  * Request URL: <URL Pointing to the application> (My Application's Cloudhub urls are located below)
  * Short Description: Subscribe to feed
  * Hints: [News/Stocks] [Search Term] *(This must be same in your config too)*
* The above step should be done for /unsubscribe and /list-subscriptions
* Once all the slash commands are created, go to **OAuth & Permissions**. Add bot scopes to the app `chat:write` as shown below
![Permissions Page](https://dev-to-uploads.s3.amazonaws.com/i/uro8re8xubj5n3ithtsm.png)
* Last thing you must to is to install the app in workspace. Click on Install App in the same page (It is in side menu). Click on **Install to Workspace**. 
* A popup opens, give a name to app and select the workspace and click OK. Give consent, then take a note of Generated Bot Token.

By this all the configuration are done. I know this is configuration heavy, but reduce UI/UX development time. This only works for few use cases anyways!

## Code Walkthrough
The subscription model contains two applications one is a subscriber and another one is a publisher

### Real-Time News Subscriber 
This API acts as an Experience API to Slack slash commands. The RAML for this API is as follows

```
#%RAML 1.0
title: Real-Time News Subscriber
version: v1

/subscribe: 
  post:
    description: Subscribe to streams from Slack via slash commands
    body:
      application/x-www-form-urlencoded:
        type: object
    responses:
      200:
        body:
          application/json:
            example: {"message": "success"}
/subscriptions: 
  post:
    description: List streams a user is subscribed to. 
    body:
      application/x-www-form-urlencoded:
        type: object
    responses:
      200:
        body:
          application/json:
            example: {"message": "success"}
/unsubscribe:
  post:
    description: Unsubscribe to streams from Slack via slash commands
    body:
      application/x-www-form-urlencoded:
        type: object
    responses:
      200:
        body:
          application/json:
            example: {"message": "success"}
```
Essentially here we have three endpoints, each listening to one Slack slash command.
* /subscribe - `/subscribe` slash command
* /subscriptions - `/list-subscriptions`
* /unsubscribe - `/unsubscribe` slash command

### Mule Flows
![Real-Time News Subscriber Flow](https://dev-to-uploads.s3.amazonaws.com/i/hipduah0juzqu15a87ai.png)
* Based on above flow, all the three endpoints are basically same. These endpoints will capture slash commands.
* From the Slash command's payload, the flow creates a body for Kafka and slack acknowledgement.
  
> Note, Slash commands should be acknowledged by our application in **3 seconds**

* When I publish message in the main thread, the slash command ack timeout and slack throws error to user, indicating the slash command failed. To avoid this the payload is published to Kafka topic asynchronously.

### Real Time News Publisher:

* In this application, the Kafka topics are listened by a consumer every 10 secs to make sure the data from topics are pulled. This is done by Kafka Message Listener, which pulls one message at a time similar to an event.
* The message received from topic will be routed based on some criteria based on operation, i.e. either subscribe or unsubscribe.
* The app doesn't need a RAML Spec as it is just a Kafka listener and a scheduler.
![Main Flow](https://dev-to-uploads.s3.amazonaws.com/i/w17lkc0skt1xbm4bmujh.png)
* If the user is subscribing, the subscription data is saved to ObjectStore, to get the stream the data on demand or by a cron job.
* Similarly if user is unsubscribing, the subscription info will be removed from ObjectStore.
* If the operation is list of subscriptions, the subscription data is fetched and send to slack. ( this operation is real time, as there is no poll needs to be run for it to finish)
* For other two operations, the scheduler will poll for every 5hrs, reads the objectstore for subscriptions, for each subscription, the call to Guardian's Content API or Alpha Vintage's Search API is called based on the stream user is subscribed to.
![Other Operations Flow](https://dev-to-uploads.s3.amazonaws.com/i/mm2jlvzqtj82iym6695o.png)
* Once data is retrieved from any API, a slack message is built and calls slack `chat.postMessage` endpoint and sends a message in slack.




