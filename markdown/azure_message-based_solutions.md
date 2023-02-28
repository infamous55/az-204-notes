---
title: Azure Message-based Solutions
created_at: 24-02-2023
tags: [azure, az-204]
---

## General characteristics of Service Bus queues

- publishers send messages to a topic, and consumers receive messages from subscriptions of the topic depending on the filter rules set
  - you can modify properties of messages when filtering by using filter action; if you want to do this only on a subset of messages in a subscription, use SQL filter expressions
- clients can receive messages without having to poll the queue
- guaranteed FIFO delivery
- the message size is up to 64 KB for the Standard tier and up to 100 MB for the Premium tier
  - the Premium tier offers fixed pricing (instead of pay-as-you-go), higher throughput, better performance, scalability, and availability
- works with the AMQP 1.0 protocol, which means it is compatible with ActiveMQ and RabbitMQ, and it is also fully compliant with the Java Message Service (JMS) 2.0 API

Note: Two benefits of using message queues are decoupling and load-leveling.

## Advanced features of Service Bus queues

- Message sessions
- Autoforwarding
- Dead-letter queues hold messages that can't be delivered to any receiver
- Scheduled delivery
- Message deferral
- Batching
- Transactions (group two or more operations in an _execution scope_)
- Filtering of messages from a topic
- Autodelete after a specified time on idle (minimum 5 minutes)
- Automatic duplicate detection
- Integration with SAS, RBAC, and Managed Identities for Azure resources
- Geo-disaster recovery

## Receive modes

- **Receive and delete**:
  - the message is marked as consumed and then returned to the consumer application
  - this works well for scenarios where you can tolerate missing messages in case of a crash
- **Peek lock**:
  - the message is locked to prevent other consumers from receiving it, returned to the application, and after the application finishes the processing, it requests the Service Bus to mark the message as being consumed
  - if the application can't process the message, it can request the Service Bus to abandon the message, in which case the message is unlocked and made available to be received again (either by the same consumer or by some other consumer)
  - there's a timeout associated with the lock, such that if the application fails to process the message before the lock timeout expires, Service Bus unlocks the message
  - this works well for applications that can't tolerate missing messages

## Message structure in Service Bus and applications

- a Service Bus message consists of a binary payload section and two sets of properties (metadata):
  - broker properties, which are system defined
  - user properties, which are defined and set by the application
- `ReplyTo` is used when a publisher expects a reply in the form of a message sent to a queue it owns
  - this works when there is only one receiver, but also when there are multiple receivers (called multicast request/reply)
- multiplexing (the ability to handle multiple concurrent operations or messages on a single connection) is achieved through sessions that are identified by matching `SessionId` values
  - `ReplyToSessionId` works similarly to `ReplyTo` in a multiplexing scenario
- payload data is serialized: the legacy SBMP protocol serializes objects with the default binary serializer, the AMQP protocol serializes objects into AMQP objects (keep in mind that they don't integrate well with HTTP), and you can also take explicit control of object serialization

Note: Applications that implement routing should do so based on user properties and not lean on the `To` property.

Note: You need to create a Service Bus messaging namespace before creating queues inside that namespace.

## General characteristics of Storage queues

- they are part of an Azure Storage account
- can store over 80 GB of messages (as opposed to Service Bus queues)
- a queue message can be up to 64 KB in size
- they work over HTTP(S)
- there are server-side logs of all transactions

Note: A time-to-live of `-1` means that the message never expires.

## Code samples

- creating a Queue service client

```csharp
QueueClient queueClient = new QueueClient(connectionString, queueName);
```

- creating a queue

```csharp
// Get the connection string from the app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to create and manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

// Create the queue
queueClient.CreateIfNotExists();
```

- inserting a message into a queue

```csharp
// Get the connection string from the app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to create and manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

// Create the queue if it doesn't already exist
queueClient.CreateIfNotExists();

if (queueClient.Exists())
{
  // Send a message to the queue
  queueClient.SendMessage(message);
}
```

- peeking at the next message

```csharp
// Get the connection string from the app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

if (queueClient.Exists())
{
  // Peek at the next message
  PeekedMessage[] peekedMessage = queueClient.PeekMessages();
}
```

- changing the contents of a queued message

```csharp
// Get the connection string from the app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

if (queueClient.Exists())
{
  // Get the message from the queue
  QueueMessage[] message = queueClient.ReceiveMessages();

  // Update the message contents
  queueClient.UpdateMessage(message[0].MessageId,
      message[0].PopReceipt,
      "Updated contents",
      TimeSpan.FromSeconds(60.0)  // Make it invisible for another 60 seconds
    );
}
```

- dequeuing the next message

```csharp
// Get the connection string from the app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

if (queueClient.Exists())
{
  // Get the next message
  QueueMessage[] retrievedMessage = queueClient.ReceiveMessages();

  // Process (i.e. print) the message in less than 30 seconds
  Console.WriteLine($"Dequeued message: '{retrievedMessage[0].Body}'");

  // Delete the message
  queueClient.DeleteMessage(retrievedMessage[0].MessageId, retrievedMessage[0].PopReceipt);
}
```

- getting the queue length

```csharp
/// Instantiate a QueueClient which will be used to manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

if (queueClient.Exists())
{
  QueueProperties properties = queueClient.GetProperties();

  // Retrieve the cached approximate message count.
  int cachedMessagesCount = properties.ApproximateMessagesCount;

  // Display the number of messages.
  Console.WriteLine($"Number of messages in queue: {cachedMessagesCount}");
}
```

- deleting a queue

```csharp
/// Get the connection string from the app settings
string connectionString = ConfigurationManager.AppSettings["StorageConnectionString"];

// Instantiate a QueueClient which will be used to manipulate the queue
QueueClient queueClient = new QueueClient(connectionString, queueName);

if (queueClient.Exists())
{
  // Delete the queue
  queueClient.Delete();
}
```

---
