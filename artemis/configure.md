# Artemis Messaging Setup for PA-iDDS-PanDA-Transformer-Harvester Integration

This document defines the **JMS destinations** (queues and topics) required for iDDS and PanDA message exchange via **Apache ActiveMQ Artemis**.

---

## Overview

We will create the following destinations:

| Destination | Type | Description |
|--------------|------|-------------|
| `/queue/tf.slices` | Queue | Receives messages from PA - consumed by iDDS |
| `/queue/panda.transformer.slices` | Queue | Receives "Transformer slice" messages from iDDS - consumed by Transformer |
| `/topic/panda.transformer` | Topic | Receives "Run end" messages from iDDS - consumed by Transformer |
| `/queue/panda.results` | Queue | Carries results from Transformer - two consumers (PA, iDDS) |
| `/queue/panda.harvester` | Queue | Receives "Adjust worker" messages from iDDS - consumed by Harvester |
| `/queue/panda.jobs` | Queue | Receives "Job request" messages from iDDS - consumed by PanDA |


Queue message types
  /queue/tf.slices:   (iDDS subscribes to it)
    "Run imminent" from PA (with run#, number of workers and so on)
    "Slice message" from PA
    "Run end" from PA
  /queue/panda.transformer.slices: (transformer subscribes to it)
    "Transformer slice" message from iDDS
  /topic/panda.transformer: (transformer subscribes to it)
    "Run end" message from iDDS
  /queue/panda.results: (The messages in this queue will have two copies. PA subscribes to /queue/panda.resutls.sub1, iDDS subscribes to /queue/panda.results.sub2)
    "Slice result" from the transformer
    "Ack worker start" from the transformer
    "Ack worker end" from the transformer
    "Worker heartbeat" from the transformer
  /queue/panda.harvester: (harvester subscribes to this one)
    "Adjust worker" message from iDDS
  /queue/panda.jobs: (panda subscribes to it)
    "new job" message from iDDS. When some panda jobs fails, iDDS can send messages to this queue to request panda to create more jobs

---

##  Step 1: Locate Broker Configuration

Edit your broker instance configuration file (usually here):

```bash
vi /opt/artemis/artemis-broker/etc/broker.xml
````

---

##  Step 2: Add Address and Queue Definitions

Inside the `<addresses>` block, add the following:

```xml
<addresses>

         <!-- User defined -->

         <!-- tf.slices: PanDA -> iDDS -->
         <address name="tf.slices">
           <anycast>
             <queue name="tf.slices" />
           </anycast>
         </address>

         <!-- panda.transformer.slices: iDDS -> Transformer -->
         <address name="panda.transformer.slices">
           <anycast>
             <queue name="panda.transformer.slices" />
           </anycast>
         </address>

         <!-- panda.transformer: iDDS -> Transformer (topic) -->
         <address name="panda.transformer">
           <multicast/>
         </address>

         <!-- panda.results: Transformer -> PA/iDDS -->
         <address name="panda.results">
           <multicast>
             <queue name="panda.results.sub1" />
             <queue name="panda.results.sub2" />
           </multicast>
         </address>

         <!-- panda.harvester: iDDS -> Harvester -->
         <address name="panda.harvester">
           <anycast>
             <queue name="panda.harvester" />
           </anycast>
         </address>

         <!-- panda.jobs: iDDS -> PanDA -->
         <address name="panda.jobs">
           <anycast>
             <queue name="panda.jobs" />
           </anycast>
         </address>

</addresses>
```

---

##  Step 3: Restart Artemis Broker

After saving your changes:

```bash
sudo systemctl restart artemis
sudo systemctl status artemis
```

Check that Artemis started cleanly and the queues were created:

```bash
/opt/artemis/artemis-broker/bin/artemis queue stat
```

You should see all six queues listed.

---

## Step 4: Verify Messaging

### Test Example (Send & Receive)

Use Artemis CLI to test sending a message to one queue:

```bash
/opt/artemis/artemis-broker/bin/artemis producer --destination queue://tf.slices --message "Run imminent test"
/opt/artemis/artemis-broker/bin/artemis consumer --destination queue://tf.slices
```

Expected output:

```
Received message: Run imminent test
```

### Test `/topic/panda.results`

When sending messages to `/topic/panda.results`, the two sub queues `/queue/panda.result.sub1` and `/queue/panda.result.sub2` should receive the same messages

```
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis producer   --destination topic://panda.results   --message "Run imminent test message"
Connection brokerURL = tcp://localhost:61616
Producer ActiveMQTopic[panda.results], thread=0 Started to calculate elapsed time ...

Producer ActiveMQTopic[panda.results], thread=0 Produced: 1000 messages
Producer ActiveMQTopic[panda.results], thread=0 Elapsed time in second : 0 s
Producer ActiveMQTopic[panda.results], thread=0 Elapsed time in milli second : 886 milli seconds
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis queue stat
Connection brokerURL = tcp://localhost:61616
|NAME                    |ADDRESS                 |CONSUMER|MESSAGE|MESSAGES|DELIVERING|MESSAGES|SCHEDULED| ROUTING |INTERNAL|
|                        |                        | COUNT  | COUNT | ADDED  |  COUNT   | ACKED  |  COUNT  |  TYPE   |        |
|$sys.mqtt.sessions      |$sys.mqtt.sessions      |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST |  true  |
|DLQ                     |DLQ                     |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|ExpiryQueue             |ExpiryQueue             |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|TEST                    |TEST                    |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.harvester         |panda.harvester         |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.jobs              |panda.jobs              |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.results.sub1      |panda.results           |   0    | 1000  |  1000  |    0     |   0    |    0    |MULTICAST| false  |
|panda.results.sub2      |panda.results           |   0    | 1000  |  1000  |    0     |   0    |    0    |MULTICAST| false  |
|panda.transformer.slices|panda.transformer.slices|   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.transformer.sub   |panda.transformer       |   0    |   0   |   0    |    0     |   0    |    0    |MULTICAST| false  |
|tf.slices               |tf.slices               |   0    |   0   |  4000  |    0     |  4000  |    0    | ANYCAST | false  |
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis consumer --destination queue://panda.results.sub1
Connection brokerURL = tcp://localhost:61616
Consumer:: filter = null
Consumer ActiveMQQueue[panda.results.sub1], thread=0 wait 3000ms until 1000 messages are consumed
Received 1000
Consumer ActiveMQQueue[panda.results.sub1], thread=0 Consumed: 1000 messages
Consumer ActiveMQQueue[panda.results.sub1], thread=0 Elapsed time in second : 0 s
Consumer ActiveMQQueue[panda.results.sub1], thread=0 Elapsed time in milli second : 151 milli seconds
Consumer ActiveMQQueue[panda.results.sub1], thread=0 Consumed: 1000 messages
Consumer ActiveMQQueue[panda.results.sub1], thread=0 Consumer thread finished
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis queue stat
Connection brokerURL = tcp://localhost:61616
|NAME                    |ADDRESS                 |CONSUMER|MESSAGE|MESSAGES|DELIVERING|MESSAGES|SCHEDULED| ROUTING |INTERNAL|
|                        |                        | COUNT  | COUNT | ADDED  |  COUNT   | ACKED  |  COUNT  |  TYPE   |        |
|$sys.mqtt.sessions      |$sys.mqtt.sessions      |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST |  true  |
|DLQ                     |DLQ                     |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|ExpiryQueue             |ExpiryQueue             |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|TEST                    |TEST                    |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.harvester         |panda.harvester         |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.jobs              |panda.jobs              |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.results.sub1      |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.results.sub2      |panda.results           |   0    | 1000  |  1000  |    0     |   0    |    0    |MULTICAST| false  |
|panda.transformer.slices|panda.transformer.slices|   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.transformer.sub   |panda.transformer       |   0    |   0   |   0    |    0     |   0    |    0    |MULTICAST| false  |
|tf.slices               |tf.slices               |   0    |   0   |  4000  |    0     |  4000  |    0    | ANYCAST | false  |
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis consumer --destination queue://panda.results.sub2
Connection brokerURL = tcp://localhost:61616
Consumer:: filter = null
Consumer ActiveMQQueue[panda.results.sub2], thread=0 wait 3000ms until 1000 messages are consumed
Received 1000
Consumer ActiveMQQueue[panda.results.sub2], thread=0 Consumed: 1000 messages
Consumer ActiveMQQueue[panda.results.sub2], thread=0 Elapsed time in second : 0 s
Consumer ActiveMQQueue[panda.results.sub2], thread=0 Elapsed time in milli second : 149 milli seconds
Consumer ActiveMQQueue[panda.results.sub2], thread=0 Consumed: 1000 messages
Consumer ActiveMQQueue[panda.results.sub2], thread=0 Consumer thread finished
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis queue stat
Connection brokerURL = tcp://localhost:61616
|NAME                    |ADDRESS                 |CONSUMER|MESSAGE|MESSAGES|DELIVERING|MESSAGES|SCHEDULED| ROUTING |INTERNAL|
|                        |                        | COUNT  | COUNT | ADDED  |  COUNT   | ACKED  |  COUNT  |  TYPE   |        |
|$sys.mqtt.sessions      |$sys.mqtt.sessions      |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST |  true  |
|DLQ                     |DLQ                     |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|ExpiryQueue             |ExpiryQueue             |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|TEST                    |TEST                    |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.harvester         |panda.harvester         |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.jobs              |panda.jobs              |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.results.sub1      |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.results.sub2      |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.transformer.slices|panda.transformer.slices|   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.transformer.sub   |panda.transformer       |   0    |   0   |   0    |    0     |   0    |    0    |MULTICAST| false  |
|tf.slices               |tf.slices               |   0    |   0   |  4000  |    0     |  4000  |    0    | ANYCAST | false  |
[root@aipanda104 artemis]# 
```

### Test `/topic/panda.transformer`

In two different terminals, create two consumers to subscribe the topic
```
/opt/artemis/artemis-broker/bin/artemis consumer --destination topic://panda.transformer
```

Two consumers are created with automatically generated names
```
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis queue stat
Connection brokerURL = tcp://localhost:61616
|NAME                     |ADDRESS                 |CONSUMER|MESSAGE|MESSAGES|DELIVERING|MESSAGES|SCHEDULED| ROUTING |INTERNAL|
|                         |                        | COUNT  | COUNT | ADDED  |  COUNT   | ACKED  |  COUNT  |  TYPE   |        |
|$sys.mqtt.sessions       |$sys.mqtt.sessions      |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST |  true  |
|DLQ                      |DLQ                     |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|ExpiryQueue              |ExpiryQueue             |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|TEST                     |TEST                    |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|d3df9e01-4cee-43b8-a1c1-5|panda.transformer       |   1    |   0   |   0    |    0     |   0    |    0    |MULTICAST| false  |
|  028848146a5            |                        |        |       |        |          |        |         |         |        |
|f03ec895-a487-4f28-bc0f-2|panda.transformer       |   1    |   0   |   0    |    0     |   0    |    0    |MULTICAST| false  |
|  0baf848dae3            |                        |        |       |        |          |        |         |         |        |
|panda.harvester          |panda.harvester         |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.jobs               |panda.jobs              |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.results.sub1       |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.results.sub2       |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.transformer.slices |panda.transformer.slices|   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.transformer.sub    |panda.transformer       |   0    |   0   |   0    |    0     |   0    |    0    |MULTICAST| false  |
|tf.slices                |tf.slices               |   0    |   0   |  4000  |    0     |  4000  |    0    | ANYCAST | false  |
```

Test to produce messages to the topic `/topic/panda.transformer`
```
 /opt/artemis/artemis-broker/bin/artemis producer   --destination topic://panda.transformer   --message "Run imminent test message"
Connection brokerURL = tcp://localhost:61616
Producer ActiveMQTopic[panda.transformer], thread=0 Started to calculate elapsed time ...

Producer ActiveMQTopic[panda.transformer], thread=0 Produced: 1000 messages
Producer ActiveMQTopic[panda.transformer], thread=0 Elapsed time in second : 1 s
Producer ActiveMQTopic[panda.transformer], thread=0 Elapsed time in milli second : 1362 milli seconds
[root@aipanda104 artemis]# /opt/artemis/artemis-broker/bin/artemis queue stat
Connection brokerURL = tcp://localhost:61616
|NAME                    |ADDRESS                 |CONSUMER|MESSAGE|MESSAGES|DELIVERING|MESSAGES|SCHEDULED| ROUTING |INTERNAL|
|                        |                        | COUNT  | COUNT | ADDED  |  COUNT   | ACKED  |  COUNT  |  TYPE   |        |
|$sys.mqtt.sessions      |$sys.mqtt.sessions      |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST |  true  |
|DLQ                     |DLQ                     |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|ExpiryQueue             |ExpiryQueue             |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|TEST                    |TEST                    |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.harvester         |panda.harvester         |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.jobs              |panda.jobs              |   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.results.sub1      |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.results.sub2      |panda.results           |   0    |   0   |  1000  |    0     |  1000  |    0    |MULTICAST| false  |
|panda.transformer.slices|panda.transformer.slices|   0    |   0   |   0    |    0     |   0    |    0    | ANYCAST | false  |
|panda.transformer.sub   |panda.transformer       |   0    | 1000  |  1000  |    0     |   0    |    0    |MULTICAST| false  |
|tf.slices               |tf.slices               |   0    |   0   |  4000  |    0     |  4000  |    0    | ANYCAST | false  |
```

---

## Step 5: Expected Message Flows

| From        | To          | Destination                       | Message Type                                                        |
| ----------- | ----------- | --------------------------------- | ------------------------------------------------------------------- |
| PA          | iDDS        | `/queue/tf.slices`                | Run imminent / Slice message / Run end                              |
| iDDS        | Transformer | `/queue/panda.transformer.slices` | Transformer slice                                                   |
| iDDS        | Transformer | `/topic/panda.transformer`        | Run end                                                             |
| Transformer | PA + iDDS   | `/queue/panda.results`            | Slice result / Ack worker start / Ack worker end / Worker heartbeat |
| iDDS        | Harvester   | `/queue/panda.harvester`          | Adjust worker                                                       |
| iDDS        | PanDA       | `/queue/panda.jobs`               | Job request for failed jobs                                         |

---

## Step 6: (Optional) Access Control

To restrict producers and consumers to specific queues, configure security roles in `broker.xml`:

```xml
<security-settings>
  <security-setting match="tf.slices">
    <permission type="createNonDurableQueue" roles="idds,pa"/>
    <permission type="send" roles="pa"/>
    <permission type="consume" roles="idds"/>
  </security-setting>
  <!-- Repeat for other queues as needed -->
</security-settings>
```

Then define these roles and users in `artemis-users.properties` and `artemis-roles.properties`.
