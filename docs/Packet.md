# MeshCore Packet Format

## Wire Format (max 255 bytes)

```
┌────────┬─────────────────┬──────────┬─────────────┬───────────────────┐
│ Header │ Transport Codes │ Path Len │    Path     │      Payload      │
│ 1 byte │ 0 or 4 bytes    │  1 byte  │ 0-64 bytes  │    0-184 bytes    │
└────────┴─────────────────┴──────────┴─────────────┴───────────────────┘
```

## Header Byte

```
┌─────────────────────────────────────────────────────────────────────┐
│  Bit 7  │  Bit 6  │ Bit 5 │ Bit 4 │ Bit 3 │ Bit 2 │ Bit 1 │ Bit 0  │
├─────────┴─────────┼───────┴───────┴───────┴───────┼───────┴────────┤
│     Version       │        Payload Type           │   Route Type   │
│     (2 bits)      │          (4 bits)             │    (2 bits)    │
└───────────────────┴───────────────────────────────┴────────────────┘
```

## Route Types (bits 0-1)

| Value | Name             | Description                                      |
|-------|------------------|--------------------------------------------------|
| 0x00  | TRANSPORT_FLOOD  | Flood routing + transport codes (4 bytes)        |
| 0x01  | FLOOD            | Flood routing, path built during travel          |
| 0x02  | DIRECT           | Direct routing, path pre-supplied                |
| 0x03  | TRANSPORT_DIRECT | Direct routing + transport codes (4 bytes)       |

## Payload Types (bits 2-5)

| Value | Name       | Description                                        |
|-------|------------|----------------------------------------------------|
| 0x00  | REQ        | Request (dest/src hash, MAC, encrypted data)       |
| 0x01  | RESPONSE   | Response to REQ or ANON_REQ                        |
| 0x02  | TXT_MSG    | Plain text message (encrypted)                     |
| 0x03  | ACK        | Simple acknowledgment                              |
| 0x04  | ADVERT     | Node advertising its identity                      |
| 0x05  | GRP_TXT    | Group text message (channel hash, MAC, encrypted)  |
| 0x06  | GRP_DATA   | Group datagram                                     |
| 0x07  | ANON_REQ   | Anonymous request (ephemeral pub_key)              |
| 0x08  | PATH       | Returned path information                          |
| 0x09  | TRACE      | Path trace, collecting SNR per hop                 |
| 0x0A  | MULTIPART  | One of a set of packets                            |
| 0x0B  | CONTROL    | Control/discovery packet                           |
| 0x0F  | RAW_CUSTOM | Custom payload (app-defined encryption)            |

## Version (bits 6-7)

| Value | Name | Description                                    |
|-------|------|------------------------------------------------|
| 0x00  | V1   | 1-byte src/dest hashes, 2-byte MAC             |
| 0x01  | V2   | FUTURE (possibly 2-byte hashes, 4-byte MAC)    |
| 0x02  | V3   | FUTURE                                         |
| 0x03  | V4   | FUTURE                                         |

## Transport Codes

When route type is TRANSPORT_FLOOD (0x00) or TRANSPORT_DIRECT (0x03), 4 bytes of transport codes are included:

```
┌────────────────┬────────────────┐
│   Code 1       │    Code 2      │  Used for zoning/filtering
│   2 bytes      │    2 bytes     │
└────────────────┴────────────────┘
```

## Payload Structures

### Peer Messages (REQ/RESPONSE/TXT_MSG/PATH)

```
┌──────────┬──────────┬─────┬────────────────────────────────────────┐
│ Dest Hash│ Src Hash │ MAC │         Encrypted Data                 │
│  1 byte  │  1 byte  │2 b  │  (timestamp + message/blob)            │
└──────────┴──────────┴─────┴────────────────────────────────────────┘
```

### Group Messages (GRP_TXT/GRP_DATA)

```
┌──────────────┬─────┬───────────────────────────────────────────────┐
│ Channel Hash │ MAC │         Encrypted Data                        │
│   1 byte     │2 b  │  (timestamp + "name: msg" or blob)            │
└──────────────┴─────┴───────────────────────────────────────────────┘
```

### Anonymous Request (ANON_REQ)

```
┌──────────┬─────────────────┬─────┬─────────────────────────────────┐
│ Dest Hash│ Ephemeral PubKey│ MAC │       Encrypted Data            │
│  1 byte  │    32 bytes     │2 b  │                                 │
└──────────┴─────────────────┴─────┴─────────────────────────────────┘
```

## Constants

| Constant           | Value     | Description                    |
|--------------------|-----------|--------------------------------|
| MAX_PACKET_PAYLOAD | 184 bytes | Maximum payload size           |
| MAX_PATH_SIZE      | 64 bytes  | Maximum path length            |
| MAX_TRANS_UNIT     | 255 bytes | Maximum transmission unit      |
| PUB_KEY_SIZE       | 32 bytes  | Public key size (Ed25519)      |
| CIPHER_MAC_SIZE    | 2 bytes   | MAC size (V1)                  |
| PATH_HASH_SIZE     | 1 byte    | Path hash size (V1)            |

## Source Reference

See `src/Packet.h` and `src/MeshCore.h` for implementation details.
