# File Chunk Connector (Source & Sink)

<img width="905" alt="image" src="https://github.com/markteehan/file-chunk-connectors/blob/main/docs/assets/Flow_20230417.png">

## _New_ -  version 2.5 (24-June-2024)
Demo Video https://www.youtube.com/@Streamsend-dp4cg/videos

Plugin packaging has changed to "streamsend": this changes the "connector.class"

New source connector configuration properties "finished.file.retention.mins" and "error.file.retention.mins" which automate cleanup of uploaded files after the number of minutes specified

New Source & Sink Connector configuration properties "topic.partitions" to inform the connectors of the partition count to distribute messages to. 
This configuration property should be set to the partition count of the topic, and it should be set for both the source and the sink connector.


## _New_ -  version 2.4 (03-May-2024)
New Source Connector configuration property "files.dir" (deprecates properties "input.path:, "error.path" and "finished.path")
New Sink Connector configuration property "files.dir" (deprecates property "output.path")
Source: new configuration properties "finished.file.retention.mins" & "error.file.retention.mins" for automated retention & cleanup of uploaded files

The file-chunk connector plugins are now available on Confluent Hub. To install the connectors, run 
```
confluent hub install markteehan/file-chunk-source:latest
confluent hub install markteehan/file-chunk-sink:latest
```

## Quick Start

### Prerequisites
- Confluent Platform
- Confluent CLI (requires separate installation)

### Create a topic
```
kafka-topics --create --topic file-chunk-topic --partitions 1 --bootstrap-server localhost:9092
```

### Create directories to upload and download files
```
cd /tmp
rm -rf upload download 
mkdir upload download
```

### Start Confluent Platform
```
confluent local services start
```

### Install the plugins
```
 confluent hub install markteehan/file-chunk-sink:latest
 confluent hub install markteehan/file-chunk-source:latest
```

### Create uploader.json

```
{
                                   "name": "uploader"
,"config":{
                                  "topic": "file-chunk-topic"
,                       "connector.class": "com.github.streamsend.file.chunk.source.ChunkSourceConnector"
 ,                            "files.dir": "/tmp/upload"
,                    "input.file.pattern": ".*"
,                             "tasks.max": "1"
,                  "file.minimum.age.ms" : "2000"
,              "binary.chunk.size.bytes" : "300000"
, "cleanup.policy.maintain.relative.path": "true"
,          "input.path.walk.recursively" : "true"
,          "finished.file.retention.mins": "10"
,                      "topic.partitions": "1"
}}

```

### Create downloader.json

```
{
                           "name": "downloader"
,"config":{
                         "topics": "file-chunk-topic"
,               "connector.class": "com.github.streamsend.file.chunk.sink.ChunkSinkConnector"
,                     "tasks.max": "1"
,                     "files.dir": "/tmp/download"
,      "binary.chunk.size.bytes" : "300000"
,         "auto.register.schemas": "false"
,                 "schema.ignore": "true"
,     "schema.generation.enabled": "false"
,"  key.converter.schemas.enable": "false"
,"value.converter.schemas.enable": "false"
,          "schema.compatibility": "NONE"
,              "topic.partitions": "1"
}}


```


### Start the connectors
If restarting, clear contents of the /tmp/upload directory first.
Using Control Center, select Connect | Connectors | Add Connector | upload connector configuration file and select the json files above, one by one.
Control Center may show a "failed" status for a few seconds before the status becomes "running".

### Test a Pipeline
The uploader will chunk and stream any files (input.file.pattern": ".*") in /tmp/upload using chunks of 1m size. Start with a small file:
```
cp /var/log/install.log /tmp/upload
```

Wait a few seconds for the file to complete processing. Examine the contents of /tmp/download/merged to confirm that "install.log" has been consumed and merged.
Diff the file to confirm that it is itentical to the uploaded file

## End of Quickstart



## TL; DR:

The Kafka Connect File Chunk Source & Sink connectors are a fancy way to send a file somewhere else.

```
Source Connector
DRONE_A179281.AVI: (size 4771930 bytes) Starting produce of 97 chunks. 
DRONE_A179281.AVI: Finished produce of 97 chunks, MD5=19b6efb31183a857e8168a6347927cc. 

Sink Connector
DRONE_A179281.AVI: Starting download of 97 chunks. 
DRONE_A179281.AVI: (size 4771930 bytes) Finished download of 97 chunks, source/target md5 matches 19b6efb31183a857e8168a6347927cc. 
```


## Whats the Why?
Three reasons for streaming file transfer:

### Files alongside events
Sometimes files accompany events: for example a modified Insurance Policy after a _claim-update_; or an image that contains a _face-recognize_. If the infrastructure for a claim-check pattern doesnt exist, then use these connectors to send the file alongside the event using the same Kafka cluster.

### Fault tolerance for edge-uploaders
Command line data send utilities require a reliable network: if connectivity is interrupted then data transfer must restart. While some utilities have improved this; in general restart-from-zero is the recovery mechanism. The file-chunk connectors use a streaming protocol which has sophisticated behaviour for network interruptions: including replay, infinite retry, inflight data, backoff and batch management. 

### Expand the role of your Kafka clusters
If your organization already uses Kafka for event-driven processing (or for logging or stream processing) then the same Kafka infrastructure can be used for streaming file transfer. The file chunk connectors are standard Kafka Connect plugins, enabling streaming pipeline patterns like fan-in (many uploaders to one downloader) and fan-out (downloaded files are mirrored to multiple locations simultaneously)



## But Kafka is not for files...
Why not? There are patterns for file-send scenarios on Kafka: sometimes they can be used; sometimes not.

### Claim check: file locators, not files

Locators generally require shared object storage accessible enterprise wide. This is common on cloud platforms, but not as common for on-premise infrastructure.

### Producers/Consumers for chunking files

File reassembly can be hard (partitions, tasks, ordering, duplication) - the Springboot/python development cost can be substantial. The file-chunk connectors handle this complexity

### BLOBs

Databases handle BLOBs (binary large objects) - so should Kafka. While a byte-encoded message that fits inside a Kafka message is unproblematic, the lack of BLOB support beyond the max.message.size makes integration with enterprise systems more problematic than necessary.


## Overview
The Kafka Connect File Chunk Source & Sink connectors watch a directory for files and read the data as new files are written to the input directory.  Each file is produced to a topic as a stream of messages after splitting the file into chunks of ```binary.chunk.size.bytes```. Messages are serialised as bytes: files can contain any content: text, logs, image, video: any binary content. The configured chunk size must be <= message.max.bytes for the Kafka cluster. 

The matching Sink connector consumes from the topic, using header metadata to reconstruct the file on a filesystem local to the Sink connector. The MD5 signature of the reconstructed target files must match the signature of the source file, otherwise the merged file is renamed with a postfix "_MD5_MISMATCH" (but processing for other files continues).  The connectors should be paired to form a complete data pipeline. Multipartition operation is supported (the Sink connector re-orders kafka messages when merging the file). These connectors do not yet support multi-task operations: they will automatically reduce the task count to one.

Subdirectories at source are recreated at target. For example consider 100 source connectors sending binary files from a local directory called queued/`hostname`.  The Sink connector reconstructs the files inside 100 subdirectories on the target machine, enabling an an additional metadata layer using subdirectory naming.

These connectors are based on the excellent [Spooldir](https://github.com/jcustenborder/kafka-connect-spooldir) connectors (created by Jeremy Custenborder) and the File-Sink connectors.


## Scenarios
It is suitable for data scenarios that include 
- file-generating edge devices (including windows clients) where a kafka connect client is preferable to custom-code uploader deployment
- sync filesystems contents to a remote server: this is only suitable for static (completed) files. It is unsuitable for mirroring open files
- edge uploader client endpoints with unreliable networking: Kafka client infinite-retries ensures eventual data delivery with no-touch intervention
- sophisticated uploader encryption - such as cipher selection
- automatic compression & decompression of file content
- Mirror files to multiple downloaders, and/or fan-in many uploaders to one Kafka cluster


## Source Connector Operation
Similar to  "spooldir", this connector monitors an input directory for new files that match an input patterns. Eligible files are split into fixed-size chunks of 
_binary.chunk.size.bytes_ which are produced to a kafka topic before renaming the file with a __FINISHED postfix flag (or an __ERROR postfix flag if any failure occurred). 
The input directory on the sending device requires sufficient headroom to duplicate the largest file, since file chunks are written to the filesystem temporarily during streaming. Files are processed one at a time: the first queued file is chunked, sent and finished; before the second file is processed; and so on. 
The connector observes & recreates subdirectories (to multiple levels): if an eligible file is created in a subdirectory "field-team-056", then the sink connector will reassemble the file in a subdirectory of the same name.

## Sink Connector Operation
The sink connector consumes events from the topic and uses header metadata to reassemble (or "merge") the files. There is one file chunk in each Kafka event. As each file chunk is consumed, the connector writes it to a local filesystem as a file. When all chunks have been consumed and written to the filesystem, then they are merged (in sequence) followed by an MD5 verification to ensure that the merged file is identical to the input file that was produced by the source connector. If the chunks are consumed in sequence, then they are appended (to a file called the "Building File"). If they are not in sequence, then each chunk is written to a separate file and each task periodically attempts to merge the chunks to the final file.


## Features
The File Chunk Source & Sink connectors include the following features:
	•	Exactly once delivery
	•	Single task operation (source and sink)
  •	Topic single- or multi- partition operation
 	•	Throughput Statistics
 
### Exactly once delivery
The File Chunk source and sink connector guarantee an exactly-once pipeline when they are run together - a combination of an at-least delivery guarantee for the source connector and duplicate handling by the sink connector. Consuming File Chunk messages from the topic using a client other than the File Chunk sink connector is not possible.

### Multiple tasks
The File Chunk connectors will support running multiple tasks with multiple topic partitions at a future date. This will enable upload and download of chunks  with multi-task parallelism, enabling files to be sent to the target faster than a single-threaded sender. Although some prior releases allowed multi-task operation, it has been disabled pending further tests as reassembly of in-order chunks outperforms reassembly of out-of-order chunks.

### Topic single- or multi- partition operation
Topic messages are keyed as the filename, so that all chunks for one file are produced to a single topic partition. Single-task operation limits scalability for multi-partition topics. Multi-task operation will be released at a future date.

### Throughput Statistics
A one-line summary of throughput for each file is logged to facilitiate tuning of partitions and chunk sizing.

```
 A_Haul_in_One.mp4: 12288 KB merged in 212 secs. Bytes/sec=57,962. 
```

## Error Handling
Set "halt.on.error=true" so that connector tasks will halt if an error is encountered. More sophisticated error handling will be added for a future release. The most likely error scenario is a full filesystem. This would prevent generation of chunk files for the source connector, or merging of file chunks for the sink connector. At present, identification and removal of finished files must be done manually - this will be automated for a future release.


## Packaging
The connectors are available packaged on Confluent Hub
 - Kafka Connect Plugins on Confluent Hub (skinny jars with dependencies) [source](https://www.confluent.io/hub/markteehan/file-chunk-source) & [sink](https://www.confluent.io/hub/markteehan/file-chunk-sink)
 

## Contacts
On technical matters, please provide feedback or questions by raising an issue against this [github repo](https://github.com/streamsend-io/file-chunk-connectors/issues)
For more specific questions (Consulting, usage scenarios etc) please email Mark Teehan (markteehan@streamsend.io)


## Installation
### Install the File Chunk Source & Sink Connector Packages
You can install this connector by manually downloading the ZIP files from Confluent Hub.
They can be installed manually by unpacking the zipfiles into the share/confluent-hub-components (or similar). The zipfiles must be unpacked.

If you have Confluent Client (or server) installed, then use the "confluent" command line utility:
confluent hub install markteehan/file-chunk-source:latest
confluent hub install markteehan/file-chunk-sink:latest


## Security (Authentication and Data Encryption)
Files are re-assembled at the sink connector as-is: the streamed files in the sink-connector "merged" directory are identical to the files in the source-connector files.dir directory.
The files selected for upload can be of any format (including any binary format): for example they can be compressed (.gz, .zip etc) or encrypted.
Topic messages (file chunks) may be optionally encrypted by setting SSL/TLS based configuration for the source and sink connectors.
Authentication and Encryption properties (using security.protocol & sasl.mechanism) must be identical for the Kafka Connect servers hosting the source and sink connectors.

The File Chunk Connectors support all Kafka Connect communication protocols, including communication with secure Kafka over TLS/SSL as well as TLS/SSL or SASL for 
authentication. 

| Security Goal         | using this encryption | and this auth     | configure this security.protocol | and this sasl.mechanism | Comments |
| --------------------- | ---------- | -------- | ----------------- | -------------- | --------------------------- |
| no encryption or auth | none       | none     | _unset_           | _unset_        | Use for dev only            |
| username/password, no encryption    | Plaintext  | SASL	    | SASL_PLAINTEXT    | _unset_        | Not recommended (insecure)  |
| username/password, traffic encrypted| TLS/SSL    | SASL     | SASL_SSL          | PLAIN          | Use for Confluent Cloud     |
| Kerberos (GSSAPI)     | TLS/SSL	   | Kerberos	| SASL_SSL          | GSSAPI         |                             |
| SASL SCRAM            | TLS/SSL	   | SCRAM	  | SASL_SSL          | SCRAM-SHA-256  |                             |

To connect to Confluent Cloud, the file chunk connectors must use SASL_SSL & PLAIN. 
Although Confluent Cloud also supports OAUTH authorisation (CP 7.2.1 or later), OAUTH is not yet supported for self-managed Kafka Connect clusters.


## Payloads
Message payloads are encoded as bytestream: there is no use of message schemas. Any Kafka client can be used to consume events created by the source connector. The accompanying file-chunk-sink connector reassembles chunks as files to a local filesystem using metadata in the message headers. This borrows from the open-source file-sink connector. 

## Ordering
Chunks are produced and consumed in order; using single-task operation and messages that are keyed as the filename.


## Limitations
These limitations are in place for the current release:
- the maximum chunk count for any single file is 100,000
- the maximum file size must fit inside the JVM of the source (and sink) machine (this is needed for MD5 verification)
- the source connector should operate on a single-node Kafka Connect cluster: this is becuase each task must be able to locate the files in the input.dir locally. Multiple source connectors, however can send to a single topic on the same Kafka cluster.
- the sink connector should operate on a single-node Kafka Connect cluster: this is because each task must be able to write files to the download subdirectories (chunks, locked, builds and merged) that are accessible to the other tasks running on the Kafka Connect server.


## License
There is no license restriction for single-task usage. Please contact Mark Teehan (markteehan@streamsend.io) for deployment of multi-task pipelines.
There is no limit on the number of single-task source/sink connectors or jobs deployed.
_Single task_ means that one Kafka Connect task produces events to a topic for each source connector, and one Kafka Connect task consumes events for each sink connector.
These plugins support single-task usage only: a _max.tasks_ > 1 is downgraded to 1 during startup.
Single Task throughput can be tuned by modifying the binary.chunk.size.bytes.

## Support
Raise an issue explaining the problem and the desired behaviour.
For consultation on feature enhancements please contact Mark Teehan (markteehan@streamsend.io).


# Troubleshooting
```org.apache.kafka.common.errors.RecordTooLargeException: The request included a message larger than the max message size the server will accept.```
The value specified for binary.chunk.size.bytes exceeds the maximum message size (message.max.bytes) for this Kafka cluster. Specify a smaller value.

```Reflections scanner could not find any classes for URLs```
or 
```Scanner SubTypesScanner was not configured```
There is an unexpected file in the plugins directory.  Do not place the plugins "zipfile" in the plugins directory - instead unzip it, so that the plugins directory contains the subdirectories markteehan-file-chunk-source-2.x (containing lib, etc and doc) and similar for the sink.



# Compatibility
The connectors are compatible with any recent Confluent Platform or Kafka release (tested against CP 7.6.x and AK 3.6.x).
It has been tested successfully using Kafka Connect deployed on Windows and on macOS.
It must be deployed on self-managed Kafka Connect clusters: they are unavailable as fully-managed connectors (or as "bring-your-own-connectors") for Confluent Cloud.
The connectors operate in single-task mode; so load balancing across multiple worker nodes for a Kafka Connect cluster is not possible. This is because the incoming (and outgoing) filesystems cannot be shared across multiple Kafka conenct servers.
Requirement for dependencies (Java, ulimits) are similar to requirements for [Confluent Platform](https://docs.confluent.io/platform/current/installation/versions-interoperability.html)


## Connect Worker Properties
Aside from common defaults, specify the following:
```
acks = ALL
max.in.flight.requests.per.connection = 1
retries = 2147483647
retry.backoff.ms = 500
producer.compression.type=none   
value.converter=org.apache.kafka.connect.converters.ByteArrayConverter
value.converter.schemas.enable=false
key.converter=org.apache.kafka.connect.converters.ByteArrayConverter
key.converter.schemas.enable=false
value.converter.schema.registry.url=http://localhost:8081
key.converter.schema.registry.url=http://localhost:8081
group.id=file-chunk-group
halt.on.error=true
```



## Source Connector configuration properties

##### `files.dir`

The directory to read files that will be processed. This directory must exist and be writable by the user running Kafka Connect.
Version 2.4: configuration properties input.path, finished.path and error.path have been deprecated. Use files.dir instead.
If the source and sink connectors are running on the same machine (for testing) then ensure that the files.dir property is not set to the same directory for both connectors.
- *Importance:* HIGH
- *Type:* STRING


##### `finished.file.retention.mins`

the duration (in minutes) before uploaded files are automatically deleted from the "files.dir" directory. The default is 60 mins. Set it to -1 to never delete uploaded files.

- *Importance:* HIGH
- *Default:* 60
- *Type:* STRING

##### `error.file.retention.min`

The duration (in minutes) before files that failed to upload are automatically deleted from the "files.dir" directory. The default is -1 (never delete)

- *Importance:* HIGH
- *Default:* -1
- *Type:* STRING


##### `input.file.pattern`

Regular expression to check input file names against. This expression must match the entire filename. 

- *Importance:* HIGH
- *Type:* STRING


##### `binary.chunk.size.bytes`

The size of each data chunk that will be produced to the topic. The size must be < Kafka cluster message.max.bytes. 
Matching files in input.directory exceeding this size will be split (chunked) into multiple smaller files of this size. 
Matching files less than this size are streamed as a single chunk.

- *Importance:* HIGH
- *Type:* INT
- *Default Value:* 1048588


##### `file.minimum.age.ms`

The amount of time in milliseconds after the file was last written to before the file can be processed. This should be set appropriatly to enable large files to be copied into the input.dir before it is processed. 

- *Importance:* HIGH
- *Type:* LONG
- *Default Value:* 5000


##### `topic`

The Kafka topic to write the data to.

- *Importance:* HIGH
- *Type:* STRING


##### `tasks.max`

Maximum number of tasks to use for this connector. For this release set tasks.max=1. If a higher value is configured, then it is automatically reset to 1. 

- *Importance:* HIGH
- *Type:* INTEGER
- *Default:* 1


##### `halt.on.error`

Stop all tasks if an error is encountered while processing input files.

- *Importance:* HIGH
- *Type:* BOOLEAN
- *Default:* true


##### `topic.partitions`

The partition count of the topic specified by "topic". The file-chunk connectors can operate in single-task mode or multi-task mode. This property is needed for efficient operation in multi-task mode. If operating in single-task mode (which is the default mode) then this property is ignored. In multi-task mode, the source connector sets message keys to a range of distinct values that matches the value of "topic.partitions" to improve distribution of data to multiple partitions, while maintaining ordering for consumers.

- *Importance:* HIGH
- *Type:* STRING
- - *Default:* 1


##### `finished.file.retention.mins`

Then number of minutes to retain a file in the input.file directory after it has been uploaded successfully. Set to -1 to disable deletion.

- *Importance:* HIGH
- *Type:* INTEGER
- - *Default:* 60


##### `error.file.retention.mins`

Then number of minutes to retain a file in the input.file directory after it has been uploaded unsuccessfully. Set to -1 to disable deletion.

- *Importance:* HIGH
- *Type:* INTEGER
- - *Default:* -1



## Sink Connector configuration properties

##### `files.dir`

The directory to write files that have been processed. This directory must exist and be writable by the user running Kafka Connect. The connector will automatically create subdirectories for builds, chunks, locked and merged. The completed files will be in the merged subdriectory. The other three subdirectories are self-managed by the connector.
If the source and sink connectors are running on the same machine (for testing) then ensure that the files.dir property is not set to the same directory for both connectors.

- *Importance:* HIGH
- *Type:* STRING


##### `topics`

The topic to consume data from a paired File Chunk Source Connector. Multiple consumers (sink connectors) configured to consume from the same topic is supported. The topic can have one, or multiple partitions. The topic should be created manually prior to startup of the source or sink connectors. A schema registry is not required, as all file-chunk events are serialized as bytestream.

- *Importance:* HIGH
- *Type:* STRING
- *Default Value:* none


##### `halt.on.error`

Stop all tasks if an error is encountered while processing file merges

- *Importance:* HIGH
- *Type:* BOOLEAN
- *Default:* true

##### `topic.partitions`

The partition count of the topic specified by "topic". The file-chunk connectors can operate in single-task mode or multi-task mode. This property is needed for efficient operation in multi-task mode. If operating in single-task mode (which is the default mode) then this property is ignored. In multi-task mode, this property should be set to the same value as the "topic.partitions" property set for the source connector.

- *Importance:* HIGH
- *Type:* STRING
- - *Default:* 1

