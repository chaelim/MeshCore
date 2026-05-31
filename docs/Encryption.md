# MeshCore Encryption

## Overview

Encryption is handled by the **MeshCore firmware** (not the app). The companion radio firmware performs all encryption and decryption. Mobile/web apps communicate with the companion radio over BLE/USB/WiFi and receive already-decrypted message data.

## Encryption Scheme

| Property | Value |
|----------|-------|
| Cipher | AES-128 (ECB mode) |
| Key | First 16 bytes of shared secret |
| MAC | HMAC-SHA256 truncated to 2 bytes |
| Order | Encrypt-then-MAC |
| Key Exchange | Ed25519 → X25519 (ECDH) |

## Encrypted Payload Format

```
┌──────────────────┬────────────────────────────────────────────────────┐
│       MAC        │              AES-128 Encrypted Data                │
│     2 bytes      │         (padded to 16-byte blocks)                 │
└──────────────────┴────────────────────────────────────────────────────┘
```

## Encryption Flow

`Utils::encryptThenMAC()` in `src/Utils.cpp`:

1. AES-128 encrypt plaintext → ciphertext
2. HMAC-SHA256(shared_secret, ciphertext) → MAC (truncated to 2 bytes)
3. Output: MAC || ciphertext

```cpp
int Utils::encryptThenMAC(const uint8_t* shared_secret, uint8_t* dest, const uint8_t* src, int src_len) {
  int enc_len = encrypt(shared_secret, dest + CIPHER_MAC_SIZE, src, src_len);

  SHA256 sha;
  sha.resetHMAC(shared_secret, PUB_KEY_SIZE);
  sha.update(dest + CIPHER_MAC_SIZE, enc_len);
  sha.finalizeHMAC(shared_secret, PUB_KEY_SIZE, dest, CIPHER_MAC_SIZE);

  return CIPHER_MAC_SIZE + enc_len;
}
```

## Decryption Flow

`Utils::MACThenDecrypt()` in `src/Utils.cpp`:

1. Extract MAC from first 2 bytes
2. Compute HMAC-SHA256(shared_secret, ciphertext)
3. Compare MACs - if mismatch, reject packet (return 0)
4. AES-128 decrypt ciphertext → plaintext

```cpp
int Utils::MACThenDecrypt(const uint8_t* shared_secret, uint8_t* dest, const uint8_t* src, int src_len) {
  if (src_len <= CIPHER_MAC_SIZE) return 0;  // invalid src bytes

  uint8_t hmac[CIPHER_MAC_SIZE];
  {
    SHA256 sha;
    sha.resetHMAC(shared_secret, PUB_KEY_SIZE);
    sha.update(src + CIPHER_MAC_SIZE, src_len - CIPHER_MAC_SIZE);
    sha.finalizeHMAC(shared_secret, PUB_KEY_SIZE, hmac, CIPHER_MAC_SIZE);
  }
  if (memcmp(hmac, src, CIPHER_MAC_SIZE) == 0) {
    return decrypt(shared_secret, dest, src + CIPHER_MAC_SIZE, src_len - CIPHER_MAC_SIZE);
  }
  return 0; // invalid HMAC
}
```

## Key Exchange

Uses Ed25519 keys converted to X25519 for ECDH key exchange:

```cpp
void LocalIdentity::calcSharedSecret(uint8_t* secret, const uint8_t* other_pub_key) const {
  ed25519_key_exchange(secret, other_pub_key, prv_key);
}
```

## Key Sources by Message Type

| Message Type | Key Source | Encrypted |
|--------------|------------|-----------|
| TXT_MSG | ECDH shared secret (my private key + peer's public key) | Yes |
| REQ | ECDH shared secret (my private key + peer's public key) | Yes |
| RESPONSE | ECDH shared secret (my private key + peer's public key) | Yes |
| PATH | ECDH shared secret (my private key + peer's public key) | Yes |
| GRP_TXT | Pre-shared channel secret (32 bytes) | Yes |
| GRP_DATA | Pre-shared channel secret (32 bytes) | Yes |
| ANON_REQ | ECDH shared secret (my private key + sender's ephemeral public key) | Yes |
| ADVERT | N/A - Ed25519 signed for authenticity | No |
| ACK | N/A | No |
| TRACE | N/A | No |
| CONTROL | N/A | No |

## Constants

From `src/MeshCore.h`:

```cpp
#define PUB_KEY_SIZE        32   // Ed25519 public key
#define PRV_KEY_SIZE        64   // Ed25519 private key
#define SIGNATURE_SIZE      64   // Ed25519 signature
#define CIPHER_KEY_SIZE     16   // AES-128 key (first 16 bytes of shared secret)
#define CIPHER_BLOCK_SIZE   16   // AES block size
#define CIPHER_MAC_SIZE      2   // Truncated HMAC-SHA256
```

## Code References

- Encryption/decryption utilities: `src/Utils.cpp:30-88`
- Key exchange: `src/Identity.cpp:95-97`
- Message decryption in mesh: `src/Mesh.cpp:140-167`
- Message encryption (createDatagram): `src/Mesh.cpp:468-490`
- Group message handling: `src/Mesh.cpp:206-229`
- Anonymous request handling: `src/Mesh.cpp:179-205`

## BLE Communication Security

### BLE Link Layer Encryption

The BLE connection between the companion radio and mobile app **is encrypted** using standard Bluetooth LE Secure Connections:

```cpp
// src/helpers/esp32/SerialBLEInterface.cpp:20-22
BLESecurity sec;
sec.setStaticPIN(pin_code);  // Default: 123456
sec.setAuthenticationMode(ESP_LE_AUTH_REQ_SC_MITM_BOND);

// Characteristics require encrypted connection
pTxCharacteristic->setAccessPermissions(ESP_GATT_PERM_READ_ENC_MITM);   // line 35
pRxCharacteristic->setAccessPermissions(ESP_GATT_PERM_WRITE_ENC_MITM);  // line 39
```

This provides:

| Feature | Description |
|---------|-------------|
| LE Secure Connections | Bluetooth 4.2+ security with ECDH key exchange |
| AES-CCM | Link layer encryption |
| MITM Protection | Man-in-the-middle attack prevention |
| PIN Pairing | Static PIN (default: 123456) |
| Bonding | Persistent pairing keys |

### Application Layer Data

The message content sent over BLE is **plaintext** (already decrypted by the firmware):

```cpp
// examples/companion_radio/MyMesh.cpp:420-423
void MyMesh::onMessageRecv(const ContactInfo &from, mesh::Packet *pkt,
                           uint32_t sender_timestamp, const char *text) {
  // 'text' is already decrypted plaintext from LoRa
  queueMessage(from, TXT_TYPE_PLAIN, pkt, sender_timestamp, NULL, 0, text);
}
```

The frame sent to the app contains:
- Response code
- SNR value
- Sender's public key prefix (6 bytes)
- Path length
- Message type
- Timestamp
- Plaintext message (already decrypted from LoRa)

### Communication Layers Summary

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MeshCore Communication Layers                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  LoRa (Over the Air)                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  AES-128 + HMAC-SHA256 encrypted payload                            │    │
│  │  Decrypted by firmware using shared secret                          │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  MeshCore Firmware (Companion Radio)                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Decrypts LoRa payload → plaintext message                          │    │
│  │  Packages plaintext into frame for app                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  BLE Link (Firmware ↔ App)                                                  │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  BLE Secure Connections (AES-CCM link encryption)                   │    │
│  │  PIN pairing (default: 123456)                                      │    │
│  │  Payload: plaintext message + metadata                              │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                              │                                              │
│                              ▼                                              │
│  Mobile/Web App                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  Receives plaintext message                                         │    │
│  │  No additional decryption needed                                    │    │
│  └─────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key point**: The app receives already-decrypted messages. All LoRa encryption/decryption happens in the firmware. The BLE link provides transport security, but there is no additional application-layer encryption between firmware and app.

### BLE Code References

- ESP32 BLE interface: `src/helpers/esp32/SerialBLEInterface.cpp`
- nRF52 BLE interface: `src/helpers/nrf52/SerialBLEInterface.cpp`
- Message receive handler: `examples/companion_radio/MyMesh.cpp:420-423`
- Frame queue to app: `examples/companion_radio/MyMesh.cpp:344-380`
