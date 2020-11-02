# Hands-on with Confluent(TM) Schema Registry


The puprose of this article is to describe how to implement and use the Confluent(tm) **Schema Registry** to serialize and deserialize *Kafka* messages with schema validation.



-----------------------------
## Contents
1. [**Introduction**](#1-introduction)
    1. [What is Schema registry?](#1.1-what-is-schema-registry)
    2. [Schema Registry Concepts](#1.2-schema-registry-concepts)
    3. [Schema Registry Architecture](#1.3-schema-registry-architecture)

2. [**Implementation**](#2-implementation)
    1. [Environment Setup](#2.1-environment-setup)
    2. [Schema Registry API](#2.2-schema-registry-api)
    3. [Code](#2.3-code)
        
    
-----------------------------
## <a id="1-introduction">1 - Introduction</a>

### <a id="1.1-what-is-schema-registry">1.1 What is Schema Registry?</a>
The **Schema Registry** component is an open source component initially developed by [Confluent](https://www.confluent.io/) (Confluent is founded by the original creators of [Apache Kafka](https://kafka.apache.org/)(r)). It's a shared repository of schemas that allows centralized management of all the schemas of a data platform and therefore better data governance. These schemas are used in the process of serialization and deserialization of messages by the clients of the platform

The Schema Registry design principle is to provide a way to tackle the challenges of managing and sharing schemas between components. In such a way that the schemas are designed to support evolution such that a consumer and producer can understand different versions of those schemas but still read all information shared between both versions and safely ignore the rest.

### <a id="1.2-schema-registry-concepts">1.2 - Schema Registry Concepts</a>

Each schemas in Schema Registry are saved under a **subject**. A **subject** can be seen as a scope in which schemas can evolve and acts (define) as a namespace in the registry.  

A *Schema* stored in Schema registry is composed of 4 fields:
 | Field | Type | Description |
  | -------- | -------- | -------- |
  | subject | String | Name of the subject that this schema is registered under |
  id | Integer | Globally unique identifier of the schema |
  version | Integer | Version of the returned schema |
  schema | String | The schema string

+ **subject**
  In order that the components which produce and consume messages use schema validation and the good one, a link have to be done between the Topic used to *publish* messages and the schema to be validated. 

  Each message published in Kafka is a key-value pair. Either the message key or the message value, or both, can be validated by a schema.

  The association between a Topic and a Schema is based on the name of the subject. Three strategies are supported:
  | Strategy | Description |
  | -------- | -------- | -------- |
  | TopicNameStrategy | Derives subject name from topic name. (This is the default.) |
  RecordNameStrategy | Derives subject name from record name, and provides a way to group logically related events that may have different data structures under a subject. |
  TopicRecordNameStrategy | Derives the subject name from topic and record name, as a way to group logically related events that may have different data structures under a subject. |

  As an example, in the default configuration (TopicNameStrategy), if you have a Topic named **Person**, to be able to perform schema validation, the subject names for the key *schema* and for the value *schema* will be: **Person-key** and **Person-value**.

  This behavior can be modified by using the following configs parameters:
   
  - confluent.key.subject.name.strategy
    Determines how to construct the subject name under which the key schema is registered with the Schema Registry.
    
  - confluent.value.subject.name.strategy
    Determines how to construct the subject name under which the value schema is registered with Schema Registry.   
            
  
  
  The possible configurations for both are:
  + io.confluent.kafka.serializers.subject.TopicNameStrategy
  + io.confluent.kafka.serializers.subject.RecordNameStrategy
  + io.confluent.kafka.serializers.subject.TopicRecordNameStrategy
  
  To choose and apply the subject naming strategy you want, you have to define and set these two parameters from the Confluent CLI, using the ```--config``` option or by code.
  
  To create a topic that uses RecordNameStrategy for the value:
  ```
  kafka-topics --create --bootstrap-server localhost:9092 \
  --replication-factor 1 --partitions 1 --topic my-other-cool-topic \
  --config confluent.value.schema.validation=true --config confluent.value.subject.name.strategy=io.confluent.kafka.serializers.subject.RecordNameStrategy
  ```       
  
  To modify a topic to use RecordNameStrategy as the key:
  ```
  kafka-configs --bootstrap-server localhost:9092 \
  --alter --entity-type topics --entity-name my-other-cool-topic \
  --add-config confluent.key.subject.name.strategy=io.confluent.kafka.serializers.subject.RecordNameStrategy 
  ```
     
+ **version**
  An important aspect of data management is schema evolution.
  
  After the initial schema is defined, applications may need to evolve it over time. When this happens, it’s critical for the downstream consumers to be able to handle data encoded with both the old and the new schema seamlessly. 

  It's **version** field role to manage this.
  
  To serve this functionnality, Schema Registry compares the new schema with previous versions of a schema, for a given subject. Comparison criteria are managed by Schema registry **Compatibility Type**.
  
  Seven compatibility types exists:
  | Compatibility Type | Changes allowed | Check against which schemas | Upgrade first |
    | -------- | -------- | -------- | -------- |
    | BACKWARD | - Delete fields </br> - Add optional fields | Last version | Consumers |
    | BACKWARD_TRANSITIVE | - Delete fields </br> - Add optional fields | All previous versions | Consumers |
    | FORWARD | - Add fields </br> - Delete optional fields | Last version | Producers |
    | FORWARD_TRANSITIVE | - Add fields </br> - Delete optional fields | All previous versions | Producers |
    | FULL | - Add optional fields </br> - Delete optional fields | Last version | Any order |
    | FULL_TRANSITIVE | - Add optional fields </br> - Delete optional fields | All previous versions | Any order |
    | NONE | - All changes are accepted | Compatibility checking disabled | Depends |
    
    Schema Registry supports multiple formats at the same time. For example, you can have Avro schemas in one subject and Protobuf schemas in another. Furthermore, both Protobuf and JSON Schema have their own compatibility rules, so you can have your Protobuf schemas evolve in a backward or forward compatible manner,
    
    This Compatibility type can be set as a Global Value for all subjects or per subject via the Rest API oper:
    ```
    PUT /config
    PUT /config/(string: subject)
    ```
  
+ **schema**
  Three formats of schemas are suppported by the Schema Registry:
  - [Avro](http://avro.apache.org/docs/current/spec.html)
    Example:
    ```
    {
     "namespace": "org.faucon.kafka.core.schema.person",
	 "type": "record",
	 "name": "Person",
	 "fields": [ {"name": "firstName","type": "string"} ]
    }
    ```

  - [JSON](https://json-schema.org/specification.html)
    Example:
    ```
    {
     "$id": "https://faucon.org/person.schema.json",
     "$schema": "http://json-schema.org/draft-07/schema#",
     "title": "Person",
     "type": "object",
     "properties": { 
         "firstName": {
             "type": "string",
             "description": "The person's first name."
         }
      }
    }
    ```

  - [Protobuf](https://developers.google.com/protocol-buffers/docs/reference/proto3-spec)
    ```
    syntax = "proto3";
    message Person {
        required string firstName = 1;
    }
    ```

  To read and write messages from the different format, Confluent(TM) provides spepcfic Kafka serializers and deserializers.

+ Schema References
  *To be completed*
  
-----------------------------
## <a id="2-">2 - Implementation</a>

*To be completed*
