

# node-agent-sdk

[![build](https://travis-ci.org/LivePersonInc/node-agent-sdk.svg?branch=master)](https://travis-ci.org/LivePersonInc/node-agent-sdk)
[![npm version](https://img.shields.io/npm/v/node-agent-sdk.svg)](https://img.shields.io/npm/v/node-agent-sdk)
[![npm downloads](https://img.shields.io/npm/dm/node-agent-sdk.svg)](https://img.shields.io/npm/dm/node-agent-sdk.svg)
[![license](https://img.shields.io/npm/l/node-agent-sdk.svg)](LICENSE)

> LivePerson Agent Messaging SDK for NodeJS

The SDK provides a simple node JS wrapper for the [LivePerson messaging API][1].

// TODO: Rebuild Index

// TODO: Links within each area that refers to other methods or events

- [Disclaimer](#disclaimer)
- [Getting Started](#getting-started)
- [Quick Start Example](#quick-start-example)
- [Example Sending Rich Content (Structured Content)](#example-sending-rich-content-structured-content)
- [API Overview](#api-overview)
  - [Agent class](#agent-class)
  - [Events](#events)
  - [Specific notifications additions](#specific-notifications-additions)
    - [MessagingEventNotification isMe() - deprecated](#messagingeventnotification-isme-deprecated)
    - [ExConversationChangeNotification getMyRole() - deprecated](#exconversationchangenotification-getmyrole-deprecated)
  - [Messaging Agent API (backend)](#messaging-agent-api-backend)
    - [reconnect()](#reconnectskiptokengeneration)
    - [dispose()](#dispose)
    - [registerRequests(arr)](#registerrequestsarr)
    - [request(type, body[, headers], callback)](#requesttype-body-headers-callback)
- [Further documentation](#further-documentation)
- [Running The Sample App](#running-the-sample-app)
- [Contributing](#contributing)

## Disclaimer
The current SDK version starts sending *MessagingEventNotification*s immediately upon connection, but this subscription will exclude some notifications.

A new major version of the SDK will be released soon in which there is no automatic subscription, and you must explicitly subscribe to these events for each conversation in order to reveive them.

In order to guarantee compatibility with future versions of the SDK, and to ensure that no notifications are missed even with the current SDK version, it is highly recommended that your bot explicitly subscribe to *MessagingEventNotification*s for all relevant conversations, as demonstrated in the [Agent-Bot](https://github.com/LivePersonInc/node-agent-sdk/tree/master/examples/agent-bot) example's [MyCoolAgent.js](https://github.com/LivePersonInc/node-agent-sdk/blob/master/examples/agent-bot/MyCoolAgent.js).

## Getting Started

### Install

- **Option 1 - npm install (does not include examples)**

   ```sh
   npm i node-agent-sdk --save
   ```

- **Option 2 - Clone this repository (includes examples)**

    ```sh
    git clone https://github.com/LivePersonInc/node-agent-sdk.git
    ```
    Run the [greeting bot example][3] (see how in [Running The Sample Apps][4]).


### Quick Start Example
```javascript
const Agent = require('node-agent-sdk').Agent;

const agent = new Agent({
    accountId: process.env.LP_ACCOUNT,
    username: process.env.LP_USER,
    password: process.env.LP_PASS
});

agent.on('connected', () => {
    console.log(`connected...`);

    // subscribe to all conversations in the account
    agent.subscribeExConversations({
        'convState': ['OPEN']
    }, (err, resp) => {
        console.log('subscribed successfully', err, resp);
    });
});

// log all conversation updates
agent.on('cqm.ExConversationChangeNotification', notificationBody => {
    console.log(JSON.stringify(notificationBody));
})
```

```sh
LP_ACCOUNT=(YourAccountNumber) LP_USER=(YourBotUsername) LP_PASS=(YourBotPassword) node index.js
```

## API Overview

### Agent class

```javascript
new Agent({
    accountId: String,  // required
    username: String,  // required for username/password authentication and OAuth1 authentication
    password: String,  // required for username/password authentication
    appKey: String, // required for OAuth1 authentication
    secret: String, // required for OAuth1 authentication
    accessToken: String, // required for OAuth1 authentication
    accessTokenSecret: String, // required for OAuth1 authentication
    token: String, // required for token authentication
    userId: String, // required for token authentication
    assertion: String, // required for SAML authentication
    csdsDomain: String, // override the CSDS domain if needed
    requestTimeout: Number, // default to 10000 milliseconds
    errorCheckInterval: Number, // defaults to 1000 milliseconds
    apiVersion: Number // Messaging API version - defaults to 2 (version 1 is not supported anymore)
});
```
#### Authentication
The Agent Messaging SDK support the following authentication methods:
- Username and password as `username` and `password`
- Barear token as `token` with user id as `userId`
- SAML assertion as `assertion`
- OAuth1 with `username`, `appkey`, `secret`, `accessToken`, and `accessTokenSecret`

### Methods

#### subscribeExConversations
This method is used to create a subscription for conversation updates. You can subscribe to all events, or to only those events pertaining to a specific agent or agents.

```javascript
agent.subscribeExConversations({
    'convState': ['OPEN']
    'agentIds': [agent.agentId] // remove this line to subscribe to all conversations instead of just the bot's conversations
}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

Callback data on success:

`{"subScriptionId":"aaaabbbb-cccc-1234-56d7-a1b2c3d4e5f6"}`

#### subscribeAgentsState
This method is used to create a subscription for Agent State updates. An event will be received whenever the bot user's state is updated.

```javascript
agent.subscribeAgentsState({}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

Callback data on success:

`{"subScriptionId":"aaaabbbb-cccc-1234-56d7-a1b2c3d4e5f6"}`

#### subscribeRoutingTasks
This method is used to create a subscription for Routing Tasks. An event will be received whenever new conversation(s) are routed to the agent. In response your bot can 'accept' the new conversation, as described below in the updateRingState method.

```javascript
agent.subscribeRoutingTasks({}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

Callback data on success:

`{"subScriptionId":"aaaabbbb-cccc-1234-56d7-a1b2c3d4e5f6"}`

#### updateRoutingTaskSubscription

// TODO

#### unsubscribeExConversations

// TODO

#### setAgentState
This method is used to set your agent's state to one of: `'ONLINE'` (can receive routing tasks for incoming conversations), `'OCCUPIED'` (can receive routing tasks for incoming transfers only), or `'AWAY'` (cannot receive routing tasks)

```javascript
agent.setAgentState({
    'availability': 'ONLINE'
}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```
Callback data on success:

`"Agent state updated successfully"`

#### getClock
This method is used to synchronize your client clock with the messaging server's clock. It can also be used as a periodic keepalive request, to ensure that your bot's connection is maintained even in periods of low activity.

```javascript
agent.getClock({}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

Callback data on success:

`{"currentTime":1513813587308}`

#### getUserProfile
This method is used to get a consumer's profile data

```javascript
agent.getUserProfile(consumerId, (e, profile) => {
    if (e) { console.error(e) }
    console.log(profile)
});
```
// TODO: example response

#### updateRingState
This method is used to update the ring state of an incoming conversation--In other words, to accept the conversation

```javascript
agent.updateRingState({
    "ringId": "someRingId",  // Ring ID received from the routing.routingTaskNotification
    "ringState": "ACCEPTED"
}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```
#### agentRequestConversation

// TODO

#### updateConversationField
This method is used to update some field of a conversation object, such as when joining a conversation as a 'MANAGER' or during a transfer when the `SKILL` is changed and the `ASSIGNED_AGENT` is removed

```javascript
agent.updateConversationField({
    'conversationId': 'conversationId/dialogId',
    'conversationField': [{
        'field': '',
        'type': '',
        '' : ''
    }]
}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

##### Example: Join conversation as manager
```javascript
agent.updateConversationField({
    'conversationId': 'conversationId/dialogId',
    'conversationField': [{
         'field': 'ParticipantsChange',
         'type': 'ADD',
         'role': 'MANAGER'
     }]
}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

##### Example: Transfer conversation to a new skill
```javascript
agent.updateConversationField({
'conversationId': 'conversationId/dialogId',
    'conversationField': [{
        {
            'field': 'ParticipantsChange',
            'type': 'REMOVE',
            'role': 'ASSIGNED_AGENT'
        },
        {
            'field': 'Skill',
            'type': 'UPDATE',
            'skill': 'targetSkillId'
        }
    ]
}, (e, resp) => {
    if (e) { console.error(e) }
    console.log(resp)
});
```

#### publishEvent
This method is used to publish an event to a conversation.
```javascript
agent.publishEvent({
    dialogId: 'conversationId/dialogId',
    event: {}
});
```

##### Example: Sending Text
```javascript
agent.publishEvent({
	dialogId: 'MY_DIALOG_ID',
	event: {
		type: 'ContentEvent',
		contentType: 'text/plain',
		message: 'hello world!'
	}
});
```

##### Example: Sending Rich Content (Structured Content)
(for more examples see [Structured Content Templates](https://developers.liveperson.com/structured-content-templates.html))
```javascript
agent.publishEvent({
	dialogId: 'MY_DIALOG_ID',
	event: {
		type: 'RichContentEvent',
		content: {
			"type": "vertical",
			"elements": [
				{
				 "type": "image",
					"url": "http://cdn.mos.cms.futurecdn.net/vkrEdZXgwP2vFa6AEQLF7f-480-80.jpg?quality=98&strip=all",
					"tooltip": "image tooltip",
					"click": {
						"actions": [
							{
								"type": "navigate",
								"name": "Navigate to store via image",
								"lo": -73.99852590,
								"la": 40.7562724
							}
						]
					}
				},
				{
					"type": "text",
					"text": "Product Name",
					"tooltip": "text tooltip",
					"style": {
						"bold": true,
						"size": "large"
					}
				},
				{
					"type": "text",
					"text": "Product description",
					"tooltip": "text tooltip"
				},
				{
					"type": "button",
					"tooltip": "button tooltip",
					"title": "Add to cart",
					"click": {
						"actions": [
							{
									"type": "link",
									"name": "Add to cart",
									"uri": "http://www.google.com"
							}
						]
					}
				}
			]
		}
	}
}, null, [{type: 'ExternalId', id: 'MY_CARD_ID'}]);  // ExternalId is how this card will be referred to in reports
```

### Events
These are events emitted by the SDK which you can listen to and react to.
```javascript
agent.on('connected', message => {
    // socket is now connected to the server
});

agent.on('notification', body => {
    // listen on all notifications
});

agent.on('ms.MessagingEventNotification', body => { // specific notification type
    // listen on notifications of the MessagingEvent type
});

agent.on('GenericSubscribeResponse', (body, requestId) => { // specific response type
    // listen on notifications of the specified type, do something with the requestId
});

agent.on('closed', reason => {
    // socket is now closed
});

agent.on('error', err => {
    // some error happened
    // might get error.code
    // if code === 401 should call reconnect
});
```

### Messaging Agent API (backend)

All request types are dynamically assigned to the object on creation.
The supported API calls are a mirror of the LiveEngage Messaging Agent API - please read the documentation carefully for full examples.

The available API calls are:

```
getClock
agentRequestConversation
subscribeExConversations
unsubscribeExConversations
updateConversationField
publishEvent
queryMessages
updateRingState
subscribeRoutingTasks
updateRoutingTaskSubscription
getUserProfile
setAgentState
subscribeAgentsState
```

#### reconnect(skipTokenGeneration)
Will reconnect the socket with the same configurations - will also regenerate token by default.  
Use if socket closes unexpectedly or on token revocation.
use `skipTokenGeneration = true` if you want to skip the token generation
Call `reconnect` on `error` with code `401`

#### dispose()
Will dispose of the connection and unregister internal events.  
Use it in order to clean the agent from memory.

#### registerRequests(arr)

You can dynamically add functionality to the SDK by registering more requests.
For example:

// TODO: Come up with an example of a function that actually doesn't exist in the SDK

```javascript
registerRequests(['.ams.AnotherTypeOfRequest']);
// ... will register the following API:
agent.anotherTypeOfRequest({/*some data*/}, (err, response) => {
    // do something
});
```


#### request(type, body[, headers], callback)

You can call any request API functionality as follows:

// TODO: Come up with an example of a function that actually doesn't exist in the SDK

```javascript
agent.request('.ams.aam.SubscribeExConversations', {
        'convState': ['OPEN']
    }, (err, resp) => {
        console.log('subscribed successfully', err, resp);
    });
```

#### agentId

You can get your agentId from the SDK using ``agent.agentId``.

### Further documentation

- [LivePerson messaging API][1]
- [LivePerson Chat SDK][2]

When creating a request through the request builder you should provide only the `body` to the sdk request method

## Running The Sample App

To run the [greeting bot example][3]:

- Provide the following `env` variables:
   - `LP_ACCOUNT` - Your LivePerson account ID
   - `LP_USER` - Your LivePerson agent username
   - `LP_PASS` - Your LivePerson agent password

- If you are consuming the Agent Messaging SDK as a dependency, switch to the
package root:

   ```sh
   cd ./node_modules/node-agent-sdk
   ```

If you are a developer, the package root is the same as the repository root.
There is therefore no need to change directories.

- Run with npm:

   ```sh
   npm start
   ```
   
### Deprecation notices
in the `cqm.ExConversationChangeNotification` the field `firstConversation` is deprecated

#### Specific notifications additions
Some notifications support helper methods to obtain the role and to identify if the message event is from "me".

##### MessagingEventNotification isMe() - *deprecated*
This method is deprecated. please use `agent.agentId` instead  
A method to understand on each change on the messaging event if it is from the agent connected right now or not.
```javascript
agent.on('ms.MessagingEventNotification', body => {
    body.changes.forEach(change => {
        change.isMe();
    });
});
```

##### ExConversationChangeNotification getMyRole() - *deprecated*
This method is deprecated. please use `agent.agentId` instead  
A method to understand on each change on the conversation change notification conversation details the current agent role in the conversation or undefined if he is not participant.
```javascript
agent.on('cqm.ExConversationChangeNotification', body => {
    body.changes.forEach(change => {
        change.result.conversationDetails.getMyRole();  
    });
});
```


## Contributing

In lieu of a formal styleguide, take care to maintain the existing coding
style. Add unit tests for any new or changed functionality, lint and test your code.

- To run the tests:

   ```sh
   npm test
   ```

- To run the [greeting bot example][3], see [Running The Sample App][4].





[1]: https://developers.liveperson.com/agent-int-api-reference.html
[2]: https://github.com/LivePersonInc/agent-sample-app
[3]: /examples/greeting-bot.js
[4]: #running-the-sample-app
