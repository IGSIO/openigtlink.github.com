---
layout: page
title: Specification > Version 3 Proposal
header: Pages
---
{% include JB/setup %}

## Background

Since the release of version 2 protocol, we have learned that the protocol has several limitations:

* While the naming convention for querying (i.e. GET_, STT_, STP_, and RTS_ prefixes) in version 2 provides a standardized way to identify query and response messages, there is still no standard way to manage multiple queries simaltaneously. This is mainly due to the lack of mechanism to reference the original query message from the response message. One workaround is to embed a unique message ID in the device names of query and response messages (e.g. "GET_STATUS_1234" and "RTS_STATUS_1234"). This approach requires the receiver process to parse the device name every time it receives a message, and reduces the actual length of device name. 
* When developers design new message exchange schemes for specific applications, they often need to attach some application-specific information to the existing data types. While it can be achieved by bundling the data message with string messages using a BIND message, it is not ideal from the performance aspect. It would make more sense to have a way to add custom 'tags' to any messages.

## Overview of Version 3 Proposal

At [Winter Project Week 2016](http://wiki.na-mic.org/Wiki/index.php/2016_Winter_Project_Week/Projects/TrackedUltrasoundStandardization) (January 5-9, 2016, Cambridge, MA), we discussed the limitations above, and potential extension to the existing protocols _with backward compatibility_ and developed a prototype description. 
At [Summer Project Week 2016](http://www.na-mic.org/Wiki/index.php/2016_Summer_Project_Week/Tracked_Ultrasound_Standardization) (June 20-26, 2016, Heidelberg, DE) the following changes were proposed:

* A new message structure. The body in the former protocol has been divided into four parts: extended header, content, meta data header, and meta data. The message now consists of the following sections:
  * Header (58 bytes)
  * Extended Header (variable length)
  * Content (variable length)
  * Meta data header (variable length)
  * Meta data (variable length)

```
        V3 message structure:
                                         GetBodySize()
                  /-------------------------/\------------------------------------------------------------\
                                             GetPackContentSize() (subclassed) 
                                                /\
                                      /--------/  \-----------\
     |____________|___________________|________________________|___________________|_______________________|
     m_Header     m_ExtendedHeader    m_Content (old m_Body)   m_MetaDataHeader    m_MetaData
                  m_Body
```

* The _header_ section has the same format as version 2 protocol, and should contain the following information:
  * The header version (the first two bytes) will be incremented to '0x0003'
  * The Body size is the total byte size for the extended header, content, meta data header, and meta data. This will allow old clients to skip the entire message and read the successive message properly.
  * The other fields are filled in the same as the previous version.

``` cpp
typedef struct {
  igtl_uint16    version;          /* protocol version number */
  char           name[IGTL_HEADER_TYPE_SIZE];       /* data type name          */
  char           device_name[IGTL_HEADER_NAME_SIZE]; /* device name             */
  igtl_uint64    timestamp;        /* time stamp message      */
  igtl_uint64    body_size;        /* size of the body        */
  igtl_uint64    crc;              /* CRC                     */
} igtl_header;
```

* The _extended header_ section contains the following fields:
  * Extended header size (2 bytes)
  * Meta data header size (2 bytes)
  * Meta data size (4 bytes)
  * Message ID (4 bytes)
 
``` cpp
typedef struct {
  igtl_uint16    extended_header_size;          /* size of extended header */
  igtl_uint16    meta_data_header_size;         /* size of the meta data header*/
  igtl_uint32    meta_data_size;                /* size of meta data */
  igtl_uint32    message_id;                    /* message id */
} igtl_extended_header;
```

* The _content_ section is equivalent to the body section in the previous version.
  * The size of *SENT* content is computed as: `IGTL_HEADER_SIZE + CalculateContentBufferSize() + sizeof(igtl_extended_header) + GetMetaDataHeaderSize() + GetMetaDataSize();`
   * See [igtl::MessageBase](https://github.com/IGSIO/OpenIGTLink/blob/master/Source/igtlMessageBase.cxx)
  * The size of *RECEIVED* content is computed as: `GetBodySize() - (extended_header_size + meta_data_header_size + meta_data_size)`
   * See [igtl::MessageBase::CalculateReceiveContentSize()](https://github.com/IGSIO/OpenIGTLink/blob/master/Source/igtlMessageBase.cxx#L508)
* The _meta\_data\_header_ sections contains tuples of `key_size`, `encodingScheme`, and `value_size` integers. The `encodingScheme` is selected from the [following list](http://www.iana.org/assignments/character-sets/character-sets.xhtml).

* The _meta\_data_ section contains pairs of `key` and `value` strings. Keys are ASCII string, while values can be stored using different encodings.

## Protocol changes
A handshaking mechanism is needed to negotiate the version capabilities of client and server during a connection. Towards this end, server implementing any OpenIGTLink protocol version greater than 3 will respond to command messages with the server's current OpenIGTLink protocol version that is implemented.

An example:
Server is v3, client is v3.
Client sends v3 COMMAND message with command name "Version".
Server recognizes COMMAND message "Version" and responds with v3 RTS_COMMAND message "Version"

Another example:
Server is v1, client is v3
Client sends v3 COMMAND message with command name "Version".
Server drops v3 message as it does not know how to handle it.
Client times out waiting for response, knows it is dealing with a v1/v2 server.

## Messaging Format

### Overall Message Format

    Bytes
    0         58            72 
    +----------+-------------+-------------------+-----------+
    |  HEADER  | EXT_HEADER  |      CONTENT      | META_DATA | 
    +----------+-------------+-------------------+-----------+


### Header

    Bytes
    0   2                       14                                      34              42              50              58
    +---+-----------------------+---------------------------------------+---------------+---------------+---------------+
    | V | TYPE                  | DEVICE_NAME                           | TIME_STAMP    | BODY_SIZE     | CRC64         |
    +---+-----------------------+---------------------------------------+---------------+---------------+---------------+

### Extended Header
    58                     60                      64                68            72    
    +----------------------+-----------------------+-----------------+-------------+
    | extended_header_size | meta_data_header_size | meta_data_size  | message_id  |
    +----------------------+-----------------------+-----------------+-------------+


### Content

The format of the contents section is message-dependent. Please see individual message definition pages. 

### Meta Data Header

    Bytes
    0             2             4                   6               10            12                  14              18
    +-------------+-------------+-------------------+---------------+-------------+-------------------+---------------+----
    | INDEX_COUNT | KEY_SIZE_0  | VALUE_ENCODING_0  | VALUE_SIZE_0  | KEY_SIZE_1  | VALUE_ENCODING_1  | VALUE_SIZE_1  | ...
    +-------------+-------------+-------------------+---------------+-------------+-------------------+---------------+----
                  |<-------------- Metadata 0 --------------------->|<--------------- Metadata 1 -------------------->|
    
                                                    INDEX_COUNT*8+2
    ----+-------------+-------------------+---------------+
    ... |KEY_SIZE_N-1 |VALUE_ENCODING_N-1 |VALUE_SIZE_N-1 |
    ----+-------------+-------------------+---------------+
        |<----------Metadata N-1 (=INDEX_COUNT)---------->|

### Meta Data

    Bytes
    +--------+---------+--------+----------+----    ----+--------+-----------+
    | KEY_0  | VALUE_0 | KEY_1  | VALUE_1  |    ...     |KEY_N-1 | VALUE_N-1 |
    +--------+---------+--------+----------+----    ----+--------+-----------+
    |<-- Metadata 0 -->|<-- Metadata 1 --->|            |<-- Metadata N-1 -->|

## New Message Types

Message Type | GET query | STT query | STP query | RTS message | Description
-------------|-----------|-----------|-----------|-------------|------------
COMMAND      | --        | --        | --        | RTS_COMMAND | Send a command and receive a response

### COMMAND
Commands are described by a _command\_header_ structure follow by an XML structure in the content part of a message.

``` cpp
typedef struct {
  igtl_uint32    commandId;        /* The unique ID of this command */
  igtl_uint8     commandName[32];  /* The name of this command */
  igtl_uint16    encoding;         /* Character encoding type as MIBenum value (defined by IANA). Default=3 */
                                   /* Please refer http://www.iana.org/assignments/character-sets for detail */
  igtl_uint32    length;           /* Length of command */
} igtl_command_header;
```

    Command message structure
      ...                 72          76            108         110       114           114 + length       ...
      ... -----------------+-------------+-----------+---------+-------------+----------------------------- ...
      ... extended_header | commandId | commandName | encoding  | length  | command XML | meta data header ...
      ... -----------------+-------------+-----------+---------+-------------+----------------------------- ...

## Acknowledgement

The OpenIGTLink v3 propsal was drafted by the following members:

* Andras Lasso (PerkLab, Queen's University)
* Tamas Ungi (PerkLab, Queen's University)
* Christian Askeland (CustusX, IGT research, SINTEF)
* Ingerid Reinertsen (CustusX, IGT research, SINTEF)
* Ole Vegard Solberg (CustusX, IGT research, SINTEF)
* Jon Eiesland (CustusX, IGT research, SINTEF)
* Janne Beate Bakeng (CustusX, IGT research, SINTEF)
* Simon Drouin (Mcgill University, Montreal, Canada)
* Junichi Tokuda (Brigham and Women's Hospital, Boston, MA)
* Steve Pieper (Isomics, Cambridge, MA)
* Adam Rankin (VASST Laboratory, Western University, Canada)
* Thomas Kirchner (MITK, DKFZ, Heidelberg, Germany)
* Janek Gröhl (MITK, DKFZ, Heidelberg, Germany)
