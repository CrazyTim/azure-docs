---
title: 'MQTT Support in Azure Event Grid'
description: 'Describes the MQTT Support in Azure Event Grid.'
ms.topic: conceptual
ms.custom: build-2023
ms.date: 05/23/2023
author: george-guirguis
ms.author: geguirgu
---
# MQTT Support in Azure Event Grid
MQTT is a publish-subscribe messaging transport protocol that was designed for constrained environments. It’s efficient, scalable, and reliable, which made it the gold standard for communication in IoT scenarios. Event Grid supports clients that publish and subscribe to messages over MQTT v3.1.1, MQTT v3.1.1 over WebSockets, MQTT v5, and MQTT v5 over WebSockets. Event Grid also supports cross MQTT version (MQTT 3.1.1 and MQTT 5) communication.

MQTT v5 has introduced many improvements over MQTT v3.1.1 to deliver a more seamless, transparent, and efficient communication. It added:
- Better error reporting.
- More transparent communication  clients through features like user properties and content type.
- More control to clients over the communication through features like message and session expiry.
- Standard important patterns like the request-response pattern.

## Connection flow:

Your MQTT clients *must* connect over TLS 1.2 or TLS 1.3. Attempts to skip this step fail with connection. 

While connecting to Event Grid, use the following ports during communication over MQTT:

- MQTT v3.1.1 and MQTT v5 on TCP port 8883
- MQTT v3.1.1 over WebSocket and MQTTv5 over WebSocket on TCP port 443.

The CONNECT packet should include the following properties:

- The ClientId field is required, and it should include the session name of the client. The session name needs to be unique across the namespace. You can use the client authentication name as the session name if each client is using one session per client. If one client is using multiple sessions, it needs to use different values for ClientId for each of its sessions.
- The Username field is required if you didn’t select a value in the  alternativeAuthenticationNameSources during namespace creation. In that case, you need to provide your client’s authentication name in the Username field. That name needs to match the authentication name provided and the value in the client’s certificate field that was specified during the client resource creation.

Learn more about [Client authentication](mqtt-client-authentication.md)

### Multi-session support

Multi-session support enables your application MQTT clients to have more scalable and reliable implementation by connecting to Event Grid with multiple active sessions at the same time. 

To create multiple sessions per client:
- Provide the Username property in the CONNECT packet to signify your client authentication name
- Provide the ClientID property in the CONNECT packet to signify the session name such as there are one or more values for the ClientID for each Username.

For example, the following combinations of Username and ClientIds in the CONNECT packet enable the client "Mgmt-application" to connect to Event Grid over three independent sessions:

- First Session:
  - Username: Mgmt-application
  - ClientId: Mgmt-Session1
- Second Session:
  - Username: Mgmt-application
  - ClientId: Mgmt-Session2
- Third Session:
  - Username: Mgmt-application
  - ClientId: Mgmt-Session3

For more information, see [How to establish multiple sessions for a single client](mqtt-establishing-multiple-sessions-per-client.md) 

#### Handling sessions:

- If a client tries to take over another client's active session by presenting its session name, its connection request will be rejected with an unauthorized error. For example, if Client B tries to connect to session 123 that is assigned at that time for client A, Client B's connection request will be rejected.
- If a client resource is deleted without ending its session, other clients won't be able to use its session name until the session expires. For example, If client B creates a session with session name 123 then client B deleted, client A won't be able to connect to session 123 until it expires.

## MQTT features
Event Grid supports the following MQTT features:

### Quality of Service (QoS)
Event Grid supports QoS 0 and 1, which define the guarantee of message delivery on PUBLISH and SUBSCRIBE packets between clients and Event Grid. QoS 0 guarantees at-most-once delivery; messages with QoS 0 aren’t acknowledged by the subscriber nor get retransmitted by the publisher. QoS 1 guarantees at-least-once delivery; messages are acknowledged by the subscriber and get retransmitted by the publisher if they didn’t get acknowledged. QoS enables your clients to control the efficiency and reliability of the communication. 
### Persistent sessions
Event Grid supports persistent sessions for MQTT v3.1.1 such that Event Grid preserves information about a client’s session in case of disconnections to ensure reliability of the communication. This information includes the client’s subscriptions and missed/ unacknowledged QoS 1 messages. Clients can configure a persistent session through setting the cleanSession flag in the CONNECT packet to false. 
### Clean start and session expiry
MQTT v5 has introduced the clean start and session expiry features as an improvement over MQTT v3.1.1 in handling session persistence. Clean Start is a feature that allows a client to start a new session with Event Grid, discarding any previous session data. Session Expiry allows a client to inform Event Grid when an inactive session is considered expired and automatically removed. In the CONNECT packet, a client can set Clean Start flag to true and/or short session expiry interval for security reasons or to avoid any potential data conflicts that may have occurred during the previous session. A client can also set a clean start to false and/or long session expiry interval to ensure the reliability and efficiency of persistent sessions.
### User properties 
Event Grid supports user properties on MQTT v5 PUBLISH packets that allow you to add custom key-value pairs in the message header to provide more context about the message. The use cases for user properties are versatile based on your needs. You can use this feature to include the purpose or origin of the message so the receiver can handle the message without parsing the payload, saving computing resources. For example, a message with a user property indicating its purpose as a "warning" could trigger different handling logic than one with the purpose of "information."
### Request-response pattern
MQTTv5 introduced fields in the MQTT PUBLISH packet header that provide context for the response message in the request-response pattern. These fields include a response topic and a correlation ID that the responder can use in the response without prior configuration. The response information enables more efficient communication for the standard request-response pattern that is used in command-and-control scenarios.
### Message expiry interval:
In MQTT v5, message expiry interval allows messages to have a configurable lifespan. The message expiry interval is defined as the time interval between the time a message is published to Event Grid and the time when the Event Grid needs to discard the message if it hasn't been delivered. This feature is useful in scenarios where messages are only valid for a certain amount of time, such as time-sensitive commands, real-time data streaming, or security alerts. By setting a message expiry interval, Event Grid can automatically remove outdated messages, ensuring that only relevant information is available to subscribers. If a message's expiry interval is set to zero, it means the message should never expire.
### Topic aliases:
In MQTT v5, topic aliases allow a client to use a shorter alias in place of the full topic name in the published message. Event Grid maintains a mapping between the topic alias and the actual topic name. This feature can save network bandwidth and reduce the size of the message header, particularly for topics with long names. It's useful in scenarios where the same topic is repeatedly published in multiple messages, such as in sensor networks. Event Grid supports up to 10 topic aliases. A client can use a Topic Alias field in the PUBLISH packet to replace the full topic name with the corresponding alias.
### Flow control
In MQTT v5, flow control refers to the mechanism for managing the rate and size of messages that a client can handle. Flow control can be configured by setting the Maximum Packet Size and Receive Maximum parameters in the CONNECT packet. The Receive Maximum parameter allows the client to limit the number of messages sent by the broker to the number of messages that the client is able to handle. The Maximum Packet Size parameter defines the maximum size of packets that the client can receive. Event Grid has a message size limit of 512 KiB. This feature ensures reliability and stability of the communication for constrained devices with limited processing speed or storage capabilities.
### Negative acknowledgments and server-initiated disconnect packet
For MQTT v5, Event Grid is able to send negative acknowledgments (NACKs) and server-initiated disconnect packets that provide the client with more information about failures for  message delivery or connection. These features help the client diagnose the reason behind a failure and take appropriate mitigating actions. Event Grid uses the reason codes that are defined in the [MQTT v5 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)

## Current limitations

Event Grid is adding more MQTT v5 and MQTT v3.1.1 features in the future to align more with the MQTT specifications. The following list details the current differences in Event Grid's MQTT support from the MQTT specifications:

### MQTTv5 current limitations

MQTT v5 currently differs from the [MQTT v5 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html) in the following ways:
- Shared Subscriptions aren't supported yet.
- Retain flag isn't supported yet.
- Will Message isn't supported yet. Receiving a CONNECT request with Will Message results in CONNACK with 0x83 (Implementation specific error).
- Maximum QoS is 1.
- Maximum Packet Size is 512 KiB
- Message ordering isn't guaranteed.
- Subscription Identifiers aren't supported.
- Assigned Client Identifiers aren't supported yet.
- The server responds to a CONNECT request with either Authentication Method or Authentication Data with a CONNACK with code 0x8C (Bad authentication method) or 0x87 (Not Authorized) respectively.
- Topic Alias Maximum is 10. The server doesn't assign any topic aliases for outgoing messages at this time. Clients can assign and use topic aliases within set limit.
- CONNACK doesn't return Response Information property even if the CONNECT request contains Request Response Information property.
- If the server receives a PUBACK from a client with non-success response code, the connection is terminated.
- Keep Alive Maximum is 1160 seconds.

### MQTTv3.1.1 current limitations

MQTT v5 currently differs from the [MQTT v3.1.1 Specification](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html) in the following ways:
- Will Message isn't supported yet. Receiving a CONNECT request with Will Message results in a connection failure.
- QoS2 and Retain Flag aren't supported yet. A publish request with a retain flag or with a QoS2 fails and closes the connection.
- Message ordering isn't guaranteed.
- Keep Alive Maximum is 1160 seconds.

## Code samples:

[This repository](https://github.com/Azure-Samples/MqttApplicationSamples/tree/main) contains C#, C, and python code samples that show how to send telemetry, send commands, and broadcast alerts. Note that the certificates created through the samples are fit for testing, but they aren't fit for production environments. 

## Next steps:

Learn more about MQTT:
- [MQTT v3.1.1 Specification](http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html)
- [MQTT v5 Specification](https://docs.oasis-open.org/mqtt/mqtt/v5.0/mqtt-v5.0.html)

Learn more about Event Grid’s MQTT support:
- [Client authentication](mqtt-client-authentication.md)
- [How to establish multiple sessions for a single client](mqtt-establishing-multiple-sessions-per-client.md) 
