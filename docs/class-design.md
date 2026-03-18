# Class Design — Kindling, Ember, Grace

> Diagram notation:
> - `<<interface>>` = pure virtual class (all methods = 0)
> - `<<abstract>>` = has some virtual methods
> - `--implements-->` = class implements an interface
> - `--extends-->` = class inherits from another class
> - `--uses-->` = class depends on another class
> - `+` = public, `-` = private, `#` = protected

---

## Kindling (Common Utilities)

### Package

```
libs/kindling/
├── include/kindling/
│   ├── Logger.h
│   ├── Config.h
│   ├── Error.h
│   └── Types.h
└── src/
    ├── Logger.cpp
    └── Config.cpp
```

### Class Diagram

```
+---------------------------------------------+
|               <<enum>> LogLevel              |
+---------------------------------------------+
| DEBUG | INFO | WARN | ERROR                  |
+---------------------------------------------+

+---------------------------------------------+
|                   Logger                     |
+---------------------------------------------+
| - level_: LogLevel                           |
+---------------------------------------------+
| + log(level: LogLevel, msg: string): void    |
| + setLevel(level: LogLevel): void            |
| + debug(msg: string): void                   |
| + info(msg: string): void                    |
| + error(msg: string): void                   |
+---------------------------------------------+

+---------------------------------------------+
|                   Config                     |
+---------------------------------------------+
| - data_: map<string, string>                 |
+---------------------------------------------+
| + load(filepath: string): void               |
| + get<T>(key: string): T                     |
| + has(key: string): bool                     |
+---------------------------------------------+
```

### Types (value objects, no inheritance)

```
+---------------------------+    +---------------------------+
|       MacAddress          |    |        IpAddress          |
+---------------------------+    +---------------------------+
| - bytes_: array<uint8, 6> |    | - bytes_: array<uint8, 4> |
+---------------------------+    +---------------------------+
| + toString(): string      |    | + toString(): string      |
| + operator==(): bool      |    | + operator==(): bool      |
| + fromString(s): Mac      |    | + fromString(s): Ip       |
+---------------------------+    +---------------------------+

Buffer = vector<uint8_t>     Port = uint16_t
```

### Error Hierarchy

```
              std::runtime_error
                     ^
                     |  --extends-->
              +------+-------+
              | KindlingError |
              +--------------+
                     ^
                     |  --extends-->
          +----------+----------+
          |                     |
  +-------+------+    +--------+-----+
  | NetworkError  |    |  ParseError  |
  +--------------+    +--------------+
```

---

## Ember (Packet Parsing)

### Package

```
libs/ember/
├── include/ember/
│   ├── IPacketParser.h       <-- public interface
│   ├── EthernetFrame.h       <-- public
│   ├── IpPacket.h            <-- public
│   ├── ArpPacket.h           <-- public
│   ├── TcpSegment.h          <-- public
│   └── UdpDatagram.h         <-- public
└── src/
    ├── EthernetFrame.cpp
    ├── IpPacket.cpp
    ├── ArpPacket.cpp
    ├── TcpSegment.cpp
    └── UdpDatagram.cpp
```

### Class Diagram

```
+-----------------------------------------------+
|       <<interface>> IPacketParser              |
+-----------------------------------------------+
| + parse(raw: span<uint8_t>): void        = 0  |
| + serialize(): Buffer                    = 0  |
| + size(): size_t                         = 0  |
+-----------------------------------------------+
         ^               ^              ^
         |               |              |
         | --implements-->              |
         |               |              |
+--------+------+ +------+------+ +----+--------+
| EthernetFrame | |  IpPacket   | |  ArpPacket  |
+---------------+ +-------------+ +-------------+
| + dst: Mac    | | + src: Ip   | | + op: uint16|
| + src: Mac    | | + dst: Ip   | | + s_mac: Mac|
| + type: u16   | | + proto: u8 | | + s_ip: Ip  |
| + payload: Buf| | + ttl: u8   | | + t_mac: Mac|
+---------------+ | + payload   | | + t_ip: Ip  |
                  +-------------+ +-------------+
         ^               ^
         |               |
         | --implements-->
         |               |
+--------+------+ +------+--------+
|  TcpSegment   | | UdpDatagram   |
+---------------+ +---------------+
| + src_port: u16| + src_port: u16|
| + dst_port: u16| + dst_port: u16|
| + seq: u32    | | + payload: Buf|
| + ack: u32    | +---------------+
| + flags: u8   |
| + payload: Buf|
+---------------+
```

### Parsing Flow

```
raw bytes
  |
  v
EthernetFrame::parse(raw)
  |
  +-- type == 0x0806 (ARP)?
  |     |
  |     v
  |   ArpPacket::parse(payload)
  |
  +-- type == 0x0800 (IPv4)?
        |
        v
      IpPacket::parse(payload)
        |
        +-- protocol == 6 (TCP)?
        |     |
        |     v
        |   TcpSegment::parse(payload)
        |
        +-- protocol == 17 (UDP)?
              |
              v
            UdpDatagram::parse(payload)
```

### Relationships

```
EthernetFrame --uses--> ArpPacket     (when type is ARP)
EthernetFrame --uses--> IpPacket      (when type is IPv4)
IpPacket      --uses--> TcpSegment    (when protocol is TCP)
IpPacket      --uses--> UdpDatagram   (when protocol is UDP)

All parsers  --uses--> MacAddress     (from kindling)
All parsers  --uses--> IpAddress      (from kindling)
All parsers  --uses--> Buffer         (from kindling)
```

---

## Grace (Network Device I/O)

### Package

```
libs/grace/
├── include/grace/
│   ├── INetDevice.h          <-- public interface
│   ├── TapDevice.h           <-- private (only minoreardtree imports)
│   ├── VethDevice.h          <-- private
│   └── RawSocket.h           <-- private
└── src/
    ├── TapDevice.cpp
    ├── VethDevice.cpp
    └── RawSocket.cpp
```

### Class Diagram

```
+-----------------------------------------------+
|         <<interface>> INetDevice               |
+-----------------------------------------------+
| + open(): void                           = 0  |
| + close(): void                          = 0  |
| + read(): Buffer                         = 0  |
| + write(data: Buffer): void              = 0  |
| + getFd(): int                           = 0  |
| + getName(): string                      = 0  |
| + ~INetDevice(): virtual                      |
+-----------------------------------------------+
         ^               ^              ^
         |               |              |
         | --implements-->              |
         |               |              |
+--------+------+ +------+------+ +----+--------+
|   TapDevice   | | VethDevice  | |  RawSocket  |
+---------------+ +-------------+ +-------------+
| - fd_: int    | | - fd_: int  | | - fd_: int  |
| - name_: str  | | - name_: str| | - iface_: st|
+---------------+ +-------------+ +-------------+
| + TapDevice(  | | + VethDevice| | + RawSocket( |
|   name: str)  | |   (name:str)| |   iface: str)|
+---------------+ +-------------+ +-------------+
| opens          | opens raw     | opens          |
| /dev/net/tun   | socket bound  | AF_PACKET      |
| with IFF_TAP   | to veth iface | SOCK_RAW       |
+---------------+ +-------------+ +-------------+
```

### RAII Lifecycle

```
construct        open()         read()/write()       ~destructor
    |               |               |                     |
    v               v               v                     v
TapDevice(name) -> opens fd -> reads/writes frames -> closes fd
                                                     (automatic)

fd_ is managed by the object. No manual close() needed.
```

### Relationships

```
TapDevice    --implements--> INetDevice
VethDevice   --implements--> INetDevice
RawSocket    --implements--> INetDevice

All devices  --uses-->       Buffer        (from kindling)
All devices  --throws-->     NetworkError   (from kindling)
```

---

## Cross-Module Dependencies

```
+-------------+
|  kindling   |  Logger, Config, Types, Error
+------+------+
       ^
       | --uses-->
       |
+------+------+         +-------------+
|    ember    | ------->|   grace     |
| IPacketParser|  (no   | INetDevice  |
| Ethernet,IP |  direct | Tap, Veth   |
| Arp,Tcp,Udp |  dep)   | RawSocket   |
+------+------+         +------+------+
       ^                        ^
       |   --uses-->            |  --uses-->
       |                        |
+------+------------------------+------+
|          minoreardtree (app)         |
|                                      |
|  wires concrete types together:      |
|  TapDevice -> read -> parse -> ...   |
+--------------------------------------+

ember and grace do NOT depend on each other.
Only the app (minoreardtree) knows both and connects them.
```
