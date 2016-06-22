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

At [Winter Project Week 2016](http://wiki.na-mic.org/Wiki/index.php/2016_Winter_Project_Week/Projects/TrackedUltrasoundStandardization) (January 5-9, 2016, Cambridge, MA), we discussed the limitations above, and potential extension to the existing protocols _with backward compatibility_. 
At [Summer Project Week 2016](http://www.na-mic.org/Wiki/index.php/2016_Summer_Project_Week/Tracked_Ultrasound_Standardization) (June 20-26, 2016, Heidelberg, DE) The following changes were proposed:

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

```
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
 
```
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
* The _meta\_data\_header_ sections contains tuples of `key_size`, `encodingScheme`, and `value_size` integers. The `encodingScheme` is selected from the [following list](http://www.iana.org/assignments/character-sets/character-sets.xhtml):

Name | MIBenum
--- | ---
US-ASCII | 3
ISO\_8859-1:1987 | 4
ISO\_8859-2:1987 | 5
ISO\_8859-3:1988 | 6
ISO\_8859-4:1988 | 7
ISO\_8859-5:1988 | 8
ISO\_8859-6:1987 | 9
ISO\_8859-7:1987 | 10
ISO\_8859-8:1988 | 11
ISO\_8859-9:1989 | 12
ISO-8859-10 | 13
ISO\_6937-2-add | 14
JIS\_X0201 | 15
JIS\_Encoding | 16
Shift\_JIS | 17
Extended\_UNIX\_Code\_Packed\_Format\_for\_Japanese | 18
Extended\_UNIX\_Code\_Fixed\_Width\_for\_Japanese | 19
BS\_4730 | 20
SEN\_850200\_C | 21
IT | 22
ES | 23
DIN\_66003 | 24
NS\_4551-1 | 25
NF\_Z\_62-010 | 26
ISO-10646-UTF-1 | 27
ISO\_646.basic:1983 | 28
INVARIANT | 29
ISO\_646.irv:1983 | 30
NATS-SEFI | 31
NATS-SEFI-ADD | 32
NATS-DANO | 33
NATS-DANO-ADD | 34
SEN\_850200\_B | 35
KS\_C\_5601-1987 | 36
ISO-2022-KR | 37
EUC-KR | 38
ISO-2022-JP | 39
ISO-2022-JP-2 | 40
JIS\_C6220-1969-jp | 41
JIS\_C6220-1969-ro | 42
PT | 43
greek7-old | 44
latin-greek | 45
NF\_Z\_62-010\_(1973) | 46
Latin-greek-1 | 47
ISO\_5427 | 48
JIS\_C6226-1978 | 49
BS\_viewdata | 50
INIS | 51
INIS-8 | 52
INIS-cyrillic | 53
ISO\_5427:1981 | 54
ISO\_5428:1980 | 55
GB\_1988-80 | 56
GB\_2312-80 | 57
NS\_4551-2 | 58
videotex-suppl | 59
PT2 | 60
ES2 | 61
MSZ\_7795.3 | 62
JIS\_C6226-1983 | 63
greek7 | 64
ASMO\_449 | 65
iso-ir-90 | 66
JIS\_C6229-1984-a | 67
JIS\_C6229-1984-b | 68
JIS\_C6229-1984-b-add | 69
JIS\_C6229-1984-hand | 70
JIS\_C6229-1984-hand-add | 71
JIS\_C6229-1984-kana | 72
ISO\_2033-1983 | 73
ANSI\_X3.110-1983 | 74
T.61-7bit | 75
T.61-8bit | 76
ECMA-cyrillic | 77
CSA\_Z243.4-1985-1 | 78
CSA\_Z243.4-1985-2 | 79
CSA\_Z243.4-1985-gr | 80
ISO\_8859-6-E | 81
ISO\_8859-6-I | 82
T.101-G2 | 83
ISO\_8859-8-E | 84
ISO\_8859-8-I | 85
CSN\_369103 | 86
JUS\_I.B1.002 | 87
IEC\_P27-1 | 88
JUS\_I.B1.003-serb | 89
JUS\_I.B1.003-mac | 90
greek-ccitt | 91
NC\_NC00-10:81 | 92
ISO\_6937-2-25 | 93
GOST\_19768-74 | 94
ISO\_8859-supp | 95
ISO\_10367-box | 96
latin-lap | 97
JIS\_X0212-1990 | 98
DS\_2089 | 99
us-dk | 100
dk-us | 101
KSC5636 | 102
UNICODE-1-1-UTF-7 | 103
ISO-2022-CN | 104
ISO-2022-CN-EXT | 105
UTF-8 | 106
ISO-8859-13 | 109
ISO-8859-14 | 110
ISO-8859-15 | 111
ISO-8859-16 | 112
GBK | 113
GB18030 | 114
OSD\_EBCDIC\_DF04\_15 | 115
OSD\_EBCDIC\_DF03\_IRV | 116
OSD\_EBCDIC\_DF04\_1 | 117
ISO-11548-1 | 118
KZ-1048 | 119
ISO-10646-UCS-2 | 1000
ISO-10646-UCS-4 | 1001
ISO-10646-UCS-Basic | 1002
ISO-10646-Unicode-Latin1 | 1003
ISO-10646-J-1 | 1004
ISO-Unicode-IBM-1261 | 1005
ISO-Unicode-IBM-1268 | 1006
ISO-Unicode-IBM-1276 | 1007
ISO-Unicode-IBM-1264 | 1008
ISO-Unicode-IBM-1265 | 1009
UNICODE-1-1 | 1010
SCSU | 1011
UTF-7 | 1012
UTF-16BE | 1013
UTF-16LE | 1014
UTF-16 | 1015
CESU-8 | 1016
UTF-32 | 1017
UTF-32BE | 1018
UTF-32LE | 1019
BOCU-1 | 1020
ISO-8859-1-Windows-3.0-Latin-1 | 2000
ISO-8859-1-Windows-3.1-Latin-1 | 2001
ISO-8859-2-Windows-Latin-2 | 2002
ISO-8859-9-Windows-Latin-5 | 2003
hp-roman8 | 2004
Adobe-Standard-Encoding | 2005
Ventura-US | 2006
Ventura-International | 2007
DEC-MCS | 2008
IBM850 | 2009
PC8-Danish-Norwegian | 2012
IBM862 | 2013
PC8-Turkish | 2014
IBM-Symbols | 2015
IBM-Thai | 2016
HP-Legal | 2017
HP-Pi-font | 2018
HP-Math8 | 2019
Adobe-Symbol-Encoding | 2020
HP-DeskTop | 2021
Ventura-Math | 2022
Microsoft-Publishing | 2023
Windows-31J | 2024
GB2312 | 2025
Big5 | 2026
macintosh | 2027
IBM037 | 2028
IBM038 | 2029
IBM273 | 2030
IBM274 | 2031
IBM275 | 2032
IBM277 | 2033
IBM278 | 2034
IBM280 | 2035
IBM281 | 2036
IBM284 | 2037
IBM285 | 2038
IBM290 | 2039
IBM297 | 2040
IBM420 | 2041
IBM423 | 2042
IBM424 | 2043
IBM437 | 2011
IBM500 | 2044
IBM851 | 2045
IBM852 | 2010
IBM855 | 2046
IBM857 | 2047
IBM860 | 2048
IBM861 | 2049
IBM863 | 2050
IBM864 | 2051
IBM865 | 2052
IBM868 | 2053
IBM869 | 2054
IBM870 | 2055
IBM871 | 2056
IBM880 | 2057
IBM891 | 2058
IBM903 | 2059
IBM904 | 2060
IBM905 | 2061
IBM918 | 2062
IBM1026 | 2063
EBCDIC-AT-DE | 2064
EBCDIC-AT-DE-A | 2065
EBCDIC-CA-FR | 2066
EBCDIC-DK-NO | 2067
EBCDIC-DK-NO-A | 2068
EBCDIC-FI-SE | 2069
EBCDIC-FI-SE-A | 2070
EBCDIC-FR | 2071
EBCDIC-IT | 2072
EBCDIC-PT | 2073
EBCDIC-ES | 2074
EBCDIC-ES-A | 2075
EBCDIC-ES-S | 2076
EBCDIC-UK | 2077
EBCDIC-US | 2078
UNKNOWN-8BIT | 2079
MNEMONIC | 2080
MNEM | 2081
VISCII | 2082
VIQR | 2083
KOI8-R | 2084
HZ-GB-2312 | 2085
IBM866 | 2086
IBM775 | 2087
KOI8-U | 2088
IBM00858 | 2089
IBM00924 | 2090
IBM01140 | 2091
IBM01141 | 2092
IBM01142 | 2093
IBM01143 | 2094
IBM01144 | 2095
IBM01145 | 2096
IBM01146 | 2097
IBM01147 | 2098
IBM01148 | 2099
IBM01149 | 2100
Big5-HKSCS | 2101
IBM1047 | 2102
PTCP154 | 2103
Amiga-1251 | 2104
KOI7-switched | 2105
BRF | 2106
TSCII | 2107
CP51932 | 2108
windows-874 | 2109
windows-1250 | 2250
windows-1251 | 2251
windows-1252 | 2252
windows-1253 | 2253
windows-1254 | 2254
windows-1255 | 2255
windows-1256 | 2256
windows-1257 | 2257
windows-1258 | 2258
TIS-620 | 2259
CP50220 | 2260

* The _meta\_data_ section contains pairs of `key` and `value` strings. Keys are ASCII string, while values can be stored using different encodings.

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
