---
title: "An Architecture for YANG-Push to Message Broker Integration"
abbrev: "YANG-Push to Message Broker Integration"
category: info

docname: draft-ietf-nmop-yang-message-broker-integration-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Network Management Operations"
keyword:
 - Automation
 - Integration
 - Data Models
venue:
  group: "Network Management Operations"
  type: "Working Group"
  mail: "nmop@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/nmop/"
  github: "ietf-wg-nmop/draft-ietf-nmop-yang-message-broker-integration"
  latest: "https://ietf-wg-nmop.github.io/draft-ietf-nmop-yang-message-broker-integration/draft-ietf-nmop-yang-message-broker-integration.html"

author:
 -
    fullname: "Thomas Graf"
    organization: Swisscom
    street: Binzring 17
    code: CH-Zuerich 8045
    country: Switzerland
    email: "thomas.graf@swisscom.com"
 -
    fullname: "Ahmed Elhassany"
    organization: Swisscom
    street: Binzring 17
    code: CH-Zuerich 8045
    country: Switzerland
    email: "ahmed.elhassany@swisscom.com"

normative:

informative:

   Kaf11:
              target: https://kafka.apache.org/
              title: "Apache Kafka"
              author:
                - name: N. Narkhede
              date: January 2011

   Con18:
              target: https://github.com/confluentinc/schema-registry/
              title: "Confluent Schema Registry"
              author:
                - name: R. Yokota
              date: December 2018

   ConDoc18:
              target: https://docs.confluent.io/platform/current/schema-registry/
              title: "Confluent Schema Registry Documentation"
              author:
                - name: R. Yokota
              date: December 2018

   LYP23:
              target: https://github.com/network-analytics/libyangpush/
              title: "libyangpush"
              author:
                - name: Z. Lin
              date: September 2023

   Yak24:
              target: https://github.com/yang-central/yangkit/
              title: "Yangkit"
              author:
                - name: F. Feng
              date: February 2024

   YLA24:
              target: https://github.com/Zephyre777/draft-lincla-netconf-yang-library-augmentation/
              title: "libyangpush"
              author:
                - name: Z. Lin
              date: March 2024

   YSR24:
              target: https://github.com/confluentinc/schema-registry-yang-format/
              title: "YANG Schema Registry Extension"
              author:
                - name: Ahmed Elhassany
              date: February 2024

   YSRPR24:
              target: https://github.com/network-analytics/draft-daisy-kafka-yang-integration/blob/main/YANG%20Schema%20registry%20integration.pdf
              title: "YANG Schema Registry Extension Progress Report"
              author:
                - name: Ahmed Elhassany
              date: February 2024

   Atl15:
              target: https://atlas.apache.org/
              title: "Apache Atlas"
              author:
                - name: Hortonworks
              date: May 2015

   Deh22:
              target: https://www.oreilly.com/library/view/data-mesh/9781492092384/
              title: "Data Mesh"
              author:
                - name: Zhamak Dehghani
              date: March 2022

   Rab07:
              target: https://rabbitmq.com/
              title: "RabbitMQ"
              author:
                - name: VMware
              date: false

   W3C.REC-xml-20081126:
              target: https://www.w3.org/TR/2008/REC-xml-20081126
              title: "Extensible Markup Language (XML) 1.0 (Fifth Edition)"
              author:
                - name: Tim Bray
                - name: Jean Paoli
                - name: C. M. Sperberg-McQueen
                - name: Eve Maler
                - name: François Yergeau
              date: November 2008
              seriesinfo:
                "W3C": World Wide Web Consortium Recommendation REC-xml-20081126

--- abstract

   This document describes the motivation and architecture of a native
   YANG-Push notifications and YANG Schema integration into a Message
   Broker and YANG Schema Registry.

--- middle

# Introduction

   Nowadays network operators are using YANG {{!RFC7950}} to model their
   configurations and obtain YANG modelled data from their networks.  It
   is well understood that plain text are initially intended for humans
   and need effort to make it machine readable due to the lack of
   semantics.  YANG modeled data is addressing most of these needs.

   Increasingly more network operators organizing their data in a Data
   Mesh {{Deh22}} where a Message Broker such as Apache Kafka {{Kaf11}} or
   RabbitMQ {{Rab07}} facilitates the exchange of messages among data
   processing components like a stream processor to filter, enrich,
   correlate or aggregate, or a time series database to store data.

   Even though YANG is intend to ease the handling of data, this promise
   has not yet been fulfilled for Network Telemetry {{?RFC9232}}.  From
   subscribing on a YANG datastore, publishing a YANG modeled
   notifications message from the network and viewing the data in a time
   series database, manual labor, such as obtaining the YANG schema from
   the network and creating a transformation or ingestion specification
   into a time series database, is needed to make a Message Broker and
   its data processing components with YANG notifications interoparable.
   Since YANG modules can change over time, for example when a router is
   being upgraded to a newer software release, this process needs to be
   adjusted contionously, leading often to errors in the data chain if
   dependencies are not properly tracked and schema changes adjusted
   simultaneously.

##  Origins of YANG-Push

   With {{?RFC3535}} the IAB set the requirements for Network Management in
   2003.  From these requirements NETCONF {{?RFC6241}}, NETCONF
   Notifications {{?RFC5277}} and RESTCONF {{?RFC8040}} have been defined to
   configure through `<edit-config>` and retrieve operational data through
   `<get>` and NETCONF notifications through `<notification>` from a YANG
   datastore on a network node.

   With YANG-Push, as defined in {{!RFC8639}}, {{?RFC8640}} and {{!RFC8641}},
   periodical and on-change subscriptions to the YANG datastore can be
   dynamically or statically configured.  When notifications are
   dynamically configured, messages are published over the initially
   established NETCONF session, while when it is statically configured
   messages are published through HTTPS-based
   {{?I-D.ietf-netconf-https-notif}} or UDP-based
   {{?I-D.ietf-netconf-udp-notif}} transport.  {{Section 3.7 of !RFC8641}}
   describes push-update messages where the YANG subscribed data is
   being published, where {{Section 2.7 of !RFC8639}} describes the
   subscription state change notifications where changes in the
   subscription are being described.

## Origins of Apache Kafka

   Apache Kafka {{Kaf11}} is a Message Broker that supports producing and
   consuming messages from so called topics.  Each topic has one or more
   partitions where messages are replicated or load balanced to scale
   out.  With the introduction of Confluent Schema Registry {{Con18}} a
   topic can contain one or more subjects.  A subject refers to a Schema
   defining the structure of the message.  The Schema then is used to
   validate messages sent through topics and are identified by a Schema
   ID.  The Schema ID is issued when the Schema is registered to the
   Confluent Schema Registry.  Once the Schema ID is obtained, it can be
   prefixed to the message with a Confluent Schema Registry compatible
   serializer.  Messages can then be validated against Schema at the
   producer or at the consumer from a topic to ensure Schema integrity
   of the message.  The type of Schema evolution scheme can be defined
   per subject, wherever non backward compatibility changes are allowed
   or not.

## Document Scope

   This document focuses on YANG-Push {{!RFC8641}} as the messaging
   protocol between the network node and the Network Telemetry {{?RFC9232}}
   data collection.  It describes the main components and the aimed
   architecture for deploying such solution in a production network.
   Then, it illustrates the integration of the YANG 1.1 {{!RFC7950}} as a
   schema modeling language into the Apache Kafka Message Broker and
   Confluent Schema Registry {{Con18}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

##  Terminology

   This document defines the following terms:

   Message Broker:
   : is an intermediary software component that translates
   messages from the formal messaging protocol of the sender to the
   formal messaging protocol of the receiver routed in topics.  Message
   brokers are elements in Data Mesh where software applications
   communicate by exchanging formally-defined messages.

   Stream Catalog:
   : provides a single point of access that allows users
   to centrally search semantics for information across a Message
   Broker.
   : Additionally it makes use of the terms defined in {{!RFC8639}}, Apache
   Kafka {{Kaf11}} and Confluent Schema Registry Documentation {{ConDoc18}}.

   The following terms are used as defined in {{!RFC8639}}:

   *  Publisher

   *  Receiver

   *  Subscription

   *  Subscription ID

   *  Event stream filter

   *  Notification message

   The following terms are used as defined in Apache Kafka Message
   Broker {{Kaf11}}:

   *  Producer

   *  Consumer

   *  Topic

   *  Partition

   The following terms are used as defined in Confluent Schema Registry
   Documentation {{ConDoc18}}:

   *  Schema

   *  Schema ID

   *  Schema Registry

   *  Subject


# Motivation

   There are four main objectives for native YANG-Push notifications and
   YANG Schema integration into a Message Broker.

##  Automatic Onboarding

   Automate the Data Mesh onboarding of newly subscribed YANG metrics.

##  Preserve Schema

   The preservation of the YANG schema, that includes the YANG data
   types as defined in {{?RFC6991}} and the nested structure of the YANG
   module, throughout the data processing chain ensures that metrics can
   be processed and visualized as they were originally intended.  Not
   only for users but also for automated closed loop operation actions.

##  Preserve Semantic Information

   {{!RFC7950}} defines in Section 7.21.3 and 7.21.4 the description and
   reference statement.  This information is intended for the user,
   describing in a human-readable fashion the meaning of a definition.
   In Data Mesh, this information can be imported from the YANG Schema
   Registry into a Stream Catalog where subjects within Message Broker
   are identifiable and searchable.  An example of a Stream Catalog is
   Apache Atlas {{Atl15}}.  It can also be applied for time series data
   visualization in a similar fashion.

##  Standardize Data Processing Integration

   Since the YANG Schema is preserved for operational metrics in the
   Message Broker, a standardization for integration between network
   data collection and stream processor or time series database is
   implied.

#  Elements of the Architecture

   The architecture consists of 6 elements.  {{Figure-1}} gives an overview
   on the workflow.

~~~~
      +------------------------------------------------------------+
      |                    Time Series Database                    |
      +------------------------------------------------------------+
                                     ^
                                     | (12) Ingest Data
                                     | According to Schema
      +------------------------------------------------------------+
      |                Time Series Database Ingestion              |
      +------------------------------------------------------------+
   (10) Get |  ^                                   ^ (9) Validate
     Schema |  |                                   | Serialized Message
            |  |                                   | Schema on Consumer
            |  |                                   |
            |  |                              +--------------------+
            |  |                              |      Message       |
            |  |                              |      Broker        |
            |  |                              +--------------------+
            |  |                                   |
            |  | (11) Issue                        | (8) Serialize
            |  |                                   | YANG-Push Message
            |  |                                   | annotated Schema
            v  | Schema             (6) Post       | ID on Producer
      +--------------------+          Schema  +--------------------+
      |       YANG         | <--------------  |  Data Collection   |
      |  Schema Registry   | -------------->  | YANG-Push Receiver |
      +--------------------+ (7) Issue        +--------------------+
                             Schema ID     (4) Get |  ^ (3) Receive
                                            Schema |  | YANG-Push
                                                   |  | Subscription
                                                   |  | Start Message
                                                   |  |   ^
                                                   |  |   |
                                                   |  |   | (5) Publish
                                                   |  |   | YANG-Push
                                                   |  |   | Message
                                                   |  |   | with
                             (1) Discover Notif.   v  |   | Subscr. ID
      +--------------------+ Capabilities     +--------------------+
      |  Manage YANG-Push  | ---------------> |   Network Node     |
      |    Subscription    | (2) Subscribe    | YANG-Push Publisher|
      +--------------------+ ---------------> +--------------------+
~~~~
{: #Figure-1 title="End to End Workflow" artwork-align="center"}

   The workflow diagram (Figure 1) describes the steps from establishing
   the YANG-Push subscription to Time Series Database ingestion.

##  YANG-Push Subscription

   With step number (1) in the workflow diagram, the YANG-Push
   notification capabilities are being discovered according to Section 3
   of {{!RFC9196}}, in with step (2) a YANG-Push subscription is according
   to {{Section 2.4 and 2.5 of !RFC8639}} dynamically or statically
   configured, and with step (3) subscription state change notifications
   are sent according to section 2.7 from the YANG-Push publisher to the
   receiver to inform which event stream filter has been applied to
   which subscription ID.

   When the YANG-Push subscription is managed dynamically, the YANG data
   is being received on the same NETCONF session where the subscription
   is being maintained.  With configured subscription the YANG data is
   sent to the YANG-Push receiver through a separate transport session.

   {{?I-D.ietf-netconf-yang-notifications-versioning}} adds the capability
   to subscribe to a specific YANG module revision or a YANG module
   which needs to be backward compatible to in step (2) and adds the
   module name, revision and revision-label information into the
   subscription state change notifications in step (3).

   {{Figure-2}} provides and example how to create a YANG-Push configured
   subscription with NETCONF in XML {{W3C.REC-xml-20081126}} with UDP-based
   {{?I-D.ietf-netconf-udp-notif}} transport.


~~~~
{::include-fold ./figures/example-establish-configured-subscription.xml}
~~~~
{: #Figure-2 title="NETCONF Example to establish configured subscription"}

   {{Figure-3}} provides an example of a JSON encoded, {{?RFC7951}},
   subscription-started state change notification message over HTTPS-
   based {{?I-D.ietf-netconf-https-notif}} or UDP-based
   {{?I-D.ietf-netconf-udp-notif}} transport with
   {{?I-D.tgraf-netconf-notif-sequencing}},
   {{?I-D.tgraf-netconf-yang-push-observation-time}} and
   {{?I-D.ietf-netconf-yang-notifications-versioning}} as extensions for
   the same subscription.

~~~~
   {
     "ietf-notification:notification": {
       "eventTime": "2023-03-25T08:30:11.22Z",
       "ietf-notification-sequencing:sysName": "example-router",
       "ietf-notification-sequencing:sequenceNumber": 1,
       "ietf-subscribed-notification:subscription-started": {
         "id": 6666,
         "ietf-yang-push:datastore": "ietf-datastores:operational",
         "ietf-yang-push:datastore-xpath-filter": "/if:interfaces",
         "ietf-yang-push-revision:revision": "2014-05-08",
         "ietf-yang-push-revision:module-name": "ietf-interfaces",
         "ietf-yang-push-revision:revision-label": "",
         "ietf-distributed-notif:message-observation-domain-id": [1,2],
         "transport": "ietf-udp-notif-transport:udp-notif",
         "encoding": "encode-json",
         "ietf-yang-push:periodic": {
           "ietf-yang-push:period": 100
         }
       }
     }
   }
~~~~
{: #Figure-3 title="JSON YANG-Push Example for a subscription-started notification message"}

##  YANG-Push Publisher

   With step number (4) in the workflow diagram, a YANG-Push push-update
   or push-change-update message, depending on wherever periodical or
   on-change subscription has been established, is sent from the YANG-
   Push publisher to the receiver according to {{Section 3.7 of !RFC8639}}.

   {{?I-D.ahuang-netconf-notif-yang}} defines the NETCONF notification
   header specified in {{?RFC5277}} in YANG to enable JSON and CBOR
   encoding.

   {{?I-D.tgraf-netconf-notif-sequencing}} adds sysName, messagePublisherId
   and sequenceNumber in the NETCONF notification header to each message
   to identify from which network node and publishing process, according
   to {{?I-D.ietf-netconf-distributed-notif}} a network node with
   distributed architecture could have multiple messagePublisherId's,
   the message has been published from.  The sequenceNumber enables to
   recognize loss from the YANG-Push publisher in step (2) down to the
   Time Series Database Ingestion in step (12).

   {{?I-D.tgraf-netconf-yang-push-observation-time}} adds observation-time
   and point-in-time in the YANG-Push push-update or push-change-update
   message. observation-time contains the timestamp and point-in-time
   when the metrics where observed.  See Section 3 of
   {{?I-D.tgraf-netconf-yang-push-observation-time}} for more details.

   {{Figure-4}} provides an example of a JSON encoded, {{?RFC7951}}, push-
   update notification message over HTTPS-based
   {{?I-D.ietf-netconf-https-notif}} or UDP-based
   {{?I-D.ietf-netconf-udp-notif}} transport with
   {{?I-D.tgraf-netconf-notif-sequencing}} and
   {{?I-D.tgraf-netconf-yang-push-observation-time}} as extensions for the
   same subscription.

~~~~
{::include-fold ./figures/push-example-push-update.xml}
~~~~
{: #Figure-4 title="JSON YANG-Push Example for a push-update notification message"}

   {{Figure-5}} provides an example of a JSON encoded, {{?RFC7951}},
   push-change-update notification message over HTTPS-based
   {{?I-D.ietf-netconf-https-notif}} or UDP-based
   {{?I-D.ietf-netconf-udp-notif}} transport with
   {{?I-D.tgraf-netconf-notif-sequencing}} and
   {{?I-D.tgraf-netconf-yang-push-observation-time}} as extensions for the
   same subscription.

~~~~
{::include-fold ./figures/push-change-update.xml}
~~~~
{: #Figure-5 title="JSON YANG-Push Example for a push-change-update notification message"}

##  YANG-Push Receiver

   For all the YANG modules and revisions of each subscription ID in the
   subscription state change notification received in step number (3) in
   the workflow diagram, all the YANG module dependencies need to be
   determined through the YANG Library {{?RFC8525}}, and then through
   NETCONF `<get-schema>` rpc calls according to {{!RFC6022}} all YANG
   modules need to be retrieved as described in step (4) in the workflow
   diagram.

   {{?I-D.lincla-netconf-yang-library-augmentation}} extends the YANG
   Library so that not only the submodule but also the augmentation list
   can be obtained.

   Figure 9 in Section 4.1 and YANG module in {{Section 5 of !RFC8641}}
   define the payload of YANG-push notifications where "datastore-
   contents" or the "value" of a "push-change-update") is "anydata".
   {{Section 7.10 of !RFC7950}}  states that anydata represents an unknown set
   of nodes that can be modeled with YANG, and the model is not known at
   design time and that the model of the unknown set of nodes can be
   signaled through another protocol.  This poses and issue in the
   schema validation of YANG-Push notifications and will be further
   clarified in point number (1) and (2) in Appendix B.

##  YANG Schema Registry

   The schema registry SHOULD support YANG as the format for defining
   schema.  For each schema registered into the schema registry, a
   schema ID is returned.  That schema ID can be used when interacting
   with the Message Broker to indicate the schema to use with the
   message.”

   Confluent Schema Registry is pluggable.  Currently Supports AVRO,
   JSON Schema and Protobuf.  The YANG support is being developed at
   {{Yak24}} as part of this architecture.  Enable to register, obtain and
   compare {{YSR24}} YANG Schemas.  One YANG Schema with all its
   augmentations is being registered per YANG-Push subscription ID. for
   each YANG Schema a locally significant Schema ID is being issued as
   described in step (7) in the workflow diagram.

~~~~
  curl -X POST -H "Content Type: application/vnd.schemaregistry.v1+json"
  -d @ietf-interfaces@2018-02-20.json
  http://localhost:8081/subjects/ietf-interfaces/
~~~~
{: #Figure-6 title="Register ietf-interfaces.yang into YANG Schema Registry"}

~~~~
   curl http://localhost:8081/subjects/ ubjects/ | jq
~~~~
{: #Figure-7 title="List all subjects YANG Schema Registry"}

~~~~
   curl http://localhost:8081/subjects/ietf-interfaces/versions
~~~~
{: #Figure-8 title="List versions of a given subject in YANG Schema Registry"}

~~~~
   curl http://localhost:8081/subjects/ietf-interfaces/versions/1
~~~~
{: #Figure-9 title="Retrieve schema of a specific subject and version in YANG Schema Registry"}

##  YANG Message Broker Producer

   The previously issued Schema ID is prefixed to the previously in
   Section 3.3 described metadata augmented YANG push push-update
   message and serialized to a Message Broker topic in step (8) of the
   workflow diagram.

##  YANG Message_Broker Consumer

   From the Message Broker topic the message is being consumed and the
   prefixed Schema ID is being used in step (10) of the workflow diagram
   to retrieve the YANG Schema to validate the Schema integrity of the
   message.

##  YANG Time Series Database Ingestion

   The time series database ingestion specifications are being derived
   with the in Section 3.6 already retrieved Schema ID and YANG-Push
   push-update messages can be now ingested and indexed into the
   database table according to their schema in step (12).

##  YANG Time Series Database

   The YANG data is being ingested in step (12)according to the
   previously defined ingestion specification and indexed with the
   timestamp defined in observation-time as defined in
   {{?I-D.tgraf-netconf-yang-push-observation-time}}.  A network operator
   is now able to query the previously subscribed YANG data.

# Implementation Status

   > Note to the RFC-Editor: Please remove this section before publishing.

   This section records the status of known implementations of the
   protocol defined by this specification at the time of posting of this
   Internet-Draft, and is based on a proposal described in {{?RFC7942}}.
   The description of implementations in this section is intended to
   assist the IETF in its decision processes in progressing drafts to
   RFCs.  Please note that the listing of any individual implementation
   here does not imply endorsement by the IETF.  Furthermore, no effort
   has been spent to verify the information presented here that was
   supplied by IETF contributors.  This is not intended as, and must not
   be construed to be, a catalog of available implementations or their
   features.  Readers are advised to note that other implementations may
   exist.

   According to {{?RFC7942}}, "this will allow reviewers and working groups
   to assign due consideration to documents that have the benefit of
   running code, which may serve as evidence of valuable experimentation
   and feedback that have made the implemented protocols more mature.
   It is up to the individual working groups to use this information as
   they see fit".

##  YANG Schema Registry Extension

   Ahmed Elhassany is developing a YANG Schema Extension in Confluent
   Schema Registry.

   The source code can be obtained here: {{YSR24}}, the progress report
   here: {{YSRPR24}}, and was validated at the IETF 117 hackathon.

##  YANG-Push Receiver Parsing Library

   Zhuoyao Lin developed as part of her internship a library to parse
   YANG-Push subscription notifications, identify YANG module
   dependencises with YANG Library {{?RFC8525}} and obtain with NETCONF
   `<get-schema>` rpc calls {{!RFC6022}} all YANG modules from YANG-Push
   publisher.

   The source code can be obtained here: {{LYP23}} and was validated at
   the IETF 117 hackathon.

##  YANG Library Augmented-by Addition

   Zhuoyao Lin implemented
   {{?I-D.lincla-netconf-yang-library-augmentation}} in order to discover
   augmented-by YANG modules in YANG Library {{?RFC8525}}.

   The source code can be obtained here: {{YLA24}} and was validated at
   the IETF 119 hackathon.

# Security Considerations

TBD


# IANA Considerations

This document has no IANA actions.


--- back

# Project Milestones

   IETF 115:

   *  Official Project Kickoff.

   *  {{!I-D.ietf-netconf-yang-notifications-versioning}} extends schema
      reference in subscription state change notification.

   IETF 116:

   *  YANG module with augmentations can be registered in Confluent
      Schema Registry with YANG extension {{Yak24}}.

   *  {{!I-D.tgraf-netconf-notif-sequencing}} extends NETCONF notification
      header with sysName, messagePublisherId and sequenceNumber.

   *  {{!I-D.tgraf-netconf-yang-push-observation-time}} extends YANG-Push
      push-update or push-change-update message with observation-time or
      state-changed-observation-time.

   *  {{!I-D.ahuang-netconf-notif-yang}} defines the NETCONF notification
      header specified in {{?RFC5277}} in YANG.

   IETF 118:

   *  All relevant YANG modules for a subscribed xpath can be determined
      through the YANG Library {{?RFC8525}} and retrieved throug NETCONF
      `<get-schema>` rpc calls according to {{!RFC6022}}.  Gap in YANG
      library addressed in
      {{!I-D.lincla-netconf-yang-library-augmentation}}.

   IETF 119:

   *  {{!I-D.aelhassany-anydata-validation}} addresses that anydata modeled
      nodes can be validated with YANG Library RFC 8525.  6WIND VSR and
      Huawei VRP YANG-Push publisher and open-source
      {{!I-D.lincla-netconf-yang-library-augmentation}} implementation
      validated at hackathon.

   IETF 120:

   *  6WIND VSR, Huawei VRP and Cisco IOS XR YANG-Push publisher and
      {{!I-D.aelhassany-anydata-validation}} implementation validated at
      hackathon.

   *  {{!I-D.tgraf-netconf-yang-push-observation-time}} merges both
      timestamps for periodical and on-change YANG-Push subscriptions
      into one observation-time timestamp and adding a decleration
      point-in-time to describe when the observation was obesreved.

#  Open Points

   Lists all current open points to be either further researched and
   clarified or tested with running code.

   Note to the RFC-Editor: Please remove this section before publishing.

   {: vspace="0"}
   Open Point 1:
   : Figure 9 in Section 4.1 and YANG module in Section 5 of {{!RFC8641}}
      defines the payload of YANG-push notifications where "datastore-
      contents" or the "value" of a "push-change-update") is "anydata".
      {{!RFC7950}} Section 7.10 states that anydata represents an unknown
      set of nodes that can be modeled with YANG, and the model is not
      known at design time and that the model of the unknown set of
      nodes can be signaled through another protocol.  How to exchange
      the anydata modeled nodes between the YANG-Push publisher and the
      receiver given that the data nodes contained in anydata subtree is
      potentially incomplete (filtered out by xpath or subtree).  How
      can a YANG-Push receiver validate the content of anydata nodes?
      {{?I-D.aelhassany-anydata-validation}} addresses this by using YANG
      Library {{?RFC8525}} as mechanism to exchange the YANG model of the
      nodes contained in anydata.

   Open Point 2:
   : The NETCONF Notification structure is defined in {{?RFC5277}} using a
      XSD Schema.  For YANG-push {{!RFC8641}}, this XSD Schema has been
      proposed using YANG 1.1 {{!RFC7950}} modeling language in
      {{!I-D.ahuang-netconf-notif-yang}}.  Examples of notifications
      encoded in XML are provided in {{Section 5 of ?RFC5277}}.  In YANG-
      JSON {{?RFC7951}}, the specification does not provide any examples on
      how notifications should be encoded.  In YANG-CBOR {{!RFC9254}},
      notifications are considered "container-like" instances and
      examples does not show consistency with XML-based and YANG-JSON
      encoding notifications.  Assumptions are being made in
      {{?I-D.ahuang-netconf-notif-yang}} providing examples of YANG-JSON
      and YANG-CBOR encoded notifications.  The notification structure
      needs consistency accross YANG encodings.  Confirm findings and
      propose how to be addressed.

   Open Point 3:
   : Test with running code wherever with
      {{?I-D.ietf-netconf-yang-notifications-versioning}} and
      {{?I-D.lincla-netconf-yang-library-augmentation}} all datastore-
      subtree-filter or datastore-xpath-filter referenced YANG modules
      and their dependencies can be fully indentified.


# Acknowledgments
{:numbered="false"}

   The authors would like to thank Yannick Buchs and Benoit Claise for
   their review and valuable comments.  Alex Huang Feng, Jean Quilbeuf
   and Zhuoyao Lin for review and contributing code and providing
   examples and inputs to the open points.
