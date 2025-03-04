# vx is a stateless proxy service with a radius-like control protocol

## The goal

Isn't it annoying having to edit config files to add or remove users, or change some routing settings? Yep, this is exactly what this project is aiming to solve.

## vlink the protocol v1

So it's pretty much the same as radius but over tcp and with some json, becuase I can't be bothered to roll a binary proto in the very first version.

### Net

The protocol consists of a tcp connection that sends short JSON-encoded messages followed by a `\r\n\r\n` sequence (yeah, just like multiplexed http/1.1).

When a node first connects to a manager it must send a hello-request, to whish the manager must respond with an approval or denial response.

Client an a server share a secret token to encrypt any sensitive data.

Both client an a server keep track of it's messages sequence IDs, so that if a server sends a message with an id of `42` the response to this message must have the same id.

#### A message

The message structure consists of the fixed fields:

- Message type
- Sequence ID
- Attributes

Example:
```go
type Message struct {
	Type int
	Sequence int
	Attributes []Attribute
}
```

There is no authenticator unlike in radius protocol, as to make it more simple. Any sensitive fields such as usernames and passwords are encrypted by default, and if you're extra paranoid, it's possible to enable full message encryption. See the `hello` message details. When a message is encrypted, its also base64 encoded, like so: `json -> aes-gcm -> base64`.

#### Message attributes

Each attribute is a simple pair of message ID (an integer, so save up the bandwidth) and it's value. Together they form an array that is the message attribute field.

Valid value types are basic JSON primitives such as `number`, `string`, `bool`.

Example:
```go
type Attribute struct {
	ID int
	Value any
}
```

#### Message encoding

Messages are encoded as JSON arrays to avoid having to duplicate same keys over and over again.

The format is as follows:

`Message` -> `[type, sequenct, [attributes...]]`

`Attribute` -> `[id, value]`

### Defined messages

- `MsgErr` = `0`; `node->manager|manager->node`; Generic error result
- `MsgNodeHello` = `1`; `node->manager`; Request node registration;
- `MsgNodeAccept` = `2`; `manager->node`; Node registration OK;
- `MsgNodeReject` = `3`; `manager->node`; Node registration failed;
- `MsgPing` = `4`; `manager->node`; Request to check if node still available (kinda duplicates the tcp stuff really but you can never be sure that a still open connection actually indicated a node being healthy)
- `MsgPong` = `5`; `node->manager`; Response to a ping request
- `MsgUserAuth` = `6`; `node->manager`; Authorize user
- `MsgUserAuthOk` = `7`; `manager->node`; User auth ok, with added data
- `MsgUserAuthFail` = `8`; `manager->node`; User auth failed
- `MsgUserNewSession` = `9`;`node->manager`; Register a new user session
- `MsgUserNewSessionOk` = `10`; `manager->node`; User session registered, with added data
- `MsgUserNewSessionFail` = `12`; `manager->node`; Unable to create a new session  
- `MsgUserTermSession` = `13`; `manager->node`; Terminate user session
- `MsgUserTermSessionOk` = `14`; `node->manager`; Session terminated successfully
- `MsgUseSessionAcct` = `15`; `node->manager`; Account user traffic

### Defined attributes

- `AttrErrMessage` = `0`; `string`; Error message text
- `AttrNodeID` = `1`; `uuid, base64`; Node ID
- `AttrUserName` = `2`; `string, AES-GCM, base64`; Authorized user name
- `AttrUserPass` = `3`; `string, AES-GCM, base64`; Authorized user password
- `AttrUserID` = `4`; `uuid, base64`; User ID
- `AttrSpeedCapRx` = `5`; `number`; User session inbound speed cap
- `AttrSpeedCapTx` = `6`; `number`; User session outbound speed cap
- `AttrProtoVersion` = `7`; `number`; vlink protocol version
- `AttrProtoSec` = `8`; `number`; Protocol security level;  
	Acceptable values:  
	- `AttrProtoSecBase` = `0`; Encrypt only sensitive data
	- `AttrProtoSecFull` = `1`; Encrypt full messages
- `AttrNodeFingerprint` = `9`; `string, base64`; Node fingerprint as `node_id+protocol_secret -> sha256 -> base64`
- `AttrUserSessionTimeout` = `10`; `number`; Optional user session timeout in seconds
- `AttrUserSessionID` = `11`; `uuid, base64`; Session id
- `AttrAuthFailReason` = `12`; `number`; Reason why auth failed  
	Acceptable values:  
	- `AttrAuthFailReasonUserNotFound` = `0`; Username not found
	- `AttrAuthFailReasonInvalidPassword` = `1`; Invalid password
- `AttrUserNewSessionFailReason` = `13`; `number`; Reason why session creation failed  
	Acceptable values:  
	- `AttrUserNewSessionFailReasonSubscriptionExpired` = `0`; Unable to create a session because user subscription has expired
	- `AttrUserNewSessionFailReasonInvalidProxyProto` = `1`; Invalid proxy protocol
- `AttrAcctRx` = `14`; `number`; Account received bytes
- `AttrAcctTx` = `15`; `number`; Account transmitted bytes
- `AttrProxyProto` = `16`; `string`; ID of the proxy protocol (http|socks|etc...)

### Message description

#### MsgErr

Can be sent as a response to any other message if an error has occured.

Attributes:
- `AttrErrMessage`

Responses: none

#### MsgNodeHello

A node sends this message as soon as it starts up to notify a manager that it's ready.

Attributes:
- `AttrProtoVersion`
- `AttrNodeID`
- `AttrNodeFingerprint`
- `AttrProtoSec`

Responses:
- `MsgNodeAccept`
- `MsgNodeReject`

#### MsgNodeAccept

Manager accepts an registers the node.

Attributes: none

Responses: none

#### MsgNodeReject

Manager rejects node registartion.

Attributes:
- `AttrErrMessage`

Responses: none

#### MsgPing

Manager requests a node to confirm it's status.

Attributes: none

Responses:
- `MsgPong`

#### MsgPong

Node responds verifying it's status.

Attributes: none

Responses: none

#### MsgUserAuth

Node requests to authenticate an incoming client

Attributes:

- `AttrUserName`
- `AttrUserPass`

Responses:
- `MsgUserAuthOk`
- `MsgUserAuthFail`

#### MsgUserAuthOk

Manager authenticates a client and returns it's data

Attributes:

- `AttrUserID`

Responses: none

#### MsgUserAuthFail

Client authentication rejected

Attributes:

- `AttrUserNewSessionFailReason`

Responses: none

#### MsgUserNewSession

Node requests to register a new client session

Attributes:
- `AttrUserID`
- `AttrProxyProto`

Responses:
- `MsgUserNewSessionOk`
- `MsgUserNewSessionFail`

#### MsgUserNewSessionOk

Created a new session OK

Attributes:
- `AttrUserSessionID`
- `AttrSpeedCapRx`
- `AttrSpeedCapTx`
- `AttrUserSessionTimeout`

Responses: none

#### MsgUserNewSessionFail

Failed to create a user session

Attributes:
- `AttrUserNewSessionFail`

Responses: none

#### MsgUserTermSession

Manager request to a node to terminate user session

Attributes:
- `AttrUserSessionID`

Responses:
- `MsgUserTermSessionOk`

#### MsgUserTermSessionOk

User session terminated successfully

Attributes:
- `AttrUserSessionID`

Responses: none

#### MsgUseSessionAcct

Account used session traffic

Attributes:
- `AttrUserSessionID`
- `AttrAcctRx`
- `AttrAcctTx`

Responses: none
