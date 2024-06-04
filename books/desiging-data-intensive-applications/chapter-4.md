## Encoding and Evolution:

- change to an application's features also requires a change to data that it stores. 
- When a data format or schema changes, a corresponding change to application code often needs to happen.But code changes in large apps cannot happen instantaneously,
    * for server-side apps we may want to perform a rolling upgrade.
    * for client-side, the user needs to do an update.
- This means, the old and new formats may potentially all coexist in the system at the same time and the system should maintain compatability in both directions:
    * Backward compatibility: Newer code can read data written by older code. 
    * forward compatability: older code can read data written by newer code. 
- Forward compatability is a bit tricker, because it requires older code to ignore additions made by a newer version of the code.
- Data exists in two representations, (i) in memory, data is kept in objects, structs, lists. (ii) when writing to a file or sending over network, we encode to sequence of bytes. The translations from in-memory to byte sequence is called encoding and the reverse is called decoding.

# Language-specific Encoding formats: 
- programming language's built-in encoding has some disadvantages, 
    * encoding is tied to particular programming language and reading the data in another language is very difficult. 
    * to restore data in the same object types, the decoding process needs to instantiate arbitrary classes. This is a source of security problem. 
    * Versioning and efficiency is often an afterthought. 

# JSON, XML and CSV Encoding formats: 
- JSON, XML and CSV are textual formats, they have some problems, 
    * In XML and CSV, we cannot distinguish between a number and a string that happens to consist of digits. JSON distinguishes strings and numbers, but it doesn't distinguish integers and floating-point numbers, and it doesn't specify a precision.
    * JSON and XML don't support binary strings.
    * CSV doesn't have any schema, so it is up to the application to define the meaning of each row and column. 

# Binary Encoding formats: 
- JSON and XML encoding user more space compared to binary format. This lead to development of binary encodings of JSON and XML, but as they don't prescribe a schema, they need to include all the object field names within the encoded data. 
- The big difference between previous formates and binary format is that there are no field names. Instead, the encoded data contains field tags, which are numbers. Fields tags are like aliases for fields - they are a compact way of saying what field we are talking about, without having to spell out the field name. 
- Apache Thrift and Protocol Buffers are binary encoding libraries. Both require a schema for any data that is encoded. 
- Thrift and Protocal buffer each come with a code generation tool that takes a schema definition and produces classes that implement the schema in various programming languages. 
- Thrift has two different binary encoding formats, BinaryProtocol and CompactProtocol. 
- in Thrift Binary Protocol, the encoded data contains field tags, which are numbers. Those are the numbers that appear in the scheam definition.  
- in Thrift Compact Protocol, the field type and tag numbers are packed into a single byte, and by using variable length integers.
- in schema of binary encoding, each field was marked either required or optional, but this makes no difference to how the field is encoded(nothing the binary data indicates whether a field is required). The difference is simply that required enabled a runtime check that fails if the field is not set. 
- an encoded record is a concatination of its encoded fields. Each field is identified by its tag number and annotated with a datatype. 
- if old code tries to read data written by new code, which includes a new field with a tag number it doesn't recognize, it can simply ignore that field(forward compatability).  
- The datatype annotation allows the parser to determine how many bytes it needs to skip. 
- the new code can always read old data(Backward compatability), because the tag numbers still have the same meaning. Only constraint is that we cannot make a new field as required.
- on removing fields, we can remove a field that is optional and can never use that tag number again. 
- Avro is a binary encoding format. In its encoded format, there is nothing to identify fields or their datatypes. The encoding simply consists of values concatenated together. 
- To parse the binary data, we go through the fields in the order that they appear in the schema and use the schema to tell us the datatype of each field. The binary data can only be decoded correctly if the code reading that data is using the exact schema as the code that wrote data. Any mismatch in the schema between the reader and writer would mean incorrectly decoded data. 
- when writing data, the app use whatever the version of the schema it knows about - this is known as the writer's schema. 
- when app wants to decode some data, is expects it to be in some schema - called reader's schema. 
- the key idea with Avro is that the writer's schema and the reader's schema don't have to be the same - they only need to be compatible. When data is decoded, the Avro library resolves the difference by looking at the writer's schema and reade's schema side by side and translating the data from writer's schema into the reader's schema. 
- to maintain compatibility, we may only add or remove a field that has a default value. 
- for the reader to know the writer's schema of the data, we can  
    1) include the writer schema in case if its large file with lot of records. 
    2) append a version number of records.  
    3) negotiate the schema version on connection setup on network transfers. 
- as Avro's schema doesn't contain any tag numbers, it is friendlier to dynamically generated schemas.

# Modes of Data flows: 
- data flows between processes via: 
    1. databases
    2. service calls
    3. asynchronous message passing.
*Databases:* 
- In databases, the process the write to db encodes the data, and the process that reads from db decodes the data. 
- When old code reads data written by new code which has a new field and update the record, i may ignore the unknown field. This needs to be handled in the encoding. The data written by old code will remain in the db for a long time(data outlives code). Schema evolution thus allows the entire db to appear as if it was encoded with a single schema, even though the underlying storage may contain records encoded with various historial versions of the schema. 
*Data flow through REST and RPC:* 
- With data flowing through services(REST or RPC),in microservices/service-oriented architecture we should expect old and new versions of server and client to be running at the same time, and so the data encoding used by servers and clients must be compatible across versions of the service API. 
*web services:* 
- two popular approaches to web services: REST and SOAP.
- REST is a design philosophy that builds upon the principles of HTTP. An API designed according to the principles of REST is called RESTful.
- SOAP is an XML-based protocol for making n/t API requests. An API of a SOAP based service is described using an XML-based language called the Web Service Description Language(WSDL).
- for evolvability, it is important that RPC clients and servers can be changed and deployed independently. We can make an assumption that all servers will be updated first and all client second. Thus we only need backward compatability on requests, and forward compatability on responses.
- The forward and backward compatability properties of an RPC scheme are inherited from whatever encoding format it uses.
- For RESTful APIs, common approaches are to use a version number in the URL or in the HTTP Accept header. For services that use API keys to identify a particular client, another option is to store a client's requested API version on the server and allow this version selection to be updated through a separate administrative interface. 
*Data flow through Message Queues:* 
- asynchronous message-passing systems are similar to RPC in that a client's request(message) is delivered to another process with low latency. They are similar to databases in that the message is not sent via a direct network connection, but goes via an intermediary called the message broker(message queue), which stores the message temporarily. 
- message brokers are used as follows: one process sends a message to a named queue or topic, and the broker ensures that the message is delivered to one or more consumers or subscribers to that queue. 
- message brokers don't enforce any particular data model- a message is just a sequence of bytes. If the encoding format is forward and backward compatible, we can deploy publishers and consumers independently. 

