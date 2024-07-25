**MoYu WeiLong V10 Ai** smartcube protocol.

# Table of Contents <!-- omit from toc -->

- [GATT](#gatt)
  - [Services](#services)
- [Packet Structure](#packet-structure)
- [Encryption](#encryption)
- [Message Types](#message-types)
  - [Cube Info (0xA1)](#cube-info-0xa1)
  - [Cube Status (Facelet State) (0xA3)](#cube-status-facelet-state-0xa3)
  - [Cube Power (0xA4)](#cube-power-0xa4)
  - [Cube Move (0xA5)](#cube-move-0xa5)
  - [Gyroscope (0xAB)](#gyroscope-0xab)

# GATT

## Services

- 0783b03e-7735-b5a0-1760-a305d2795cb0 (**Main Cube Service**)
  - **Characteristics**:
    - 0783b03e-7735-b5a0-1760-a305d2795cb1 (**NOTIFY**)
    - 0783b03e-7735-b5a0-1760-a305d2795cb2 (**WRITE, WRITE NO RESPONSE**)
- 02f00000-0000-0000-0000-00000000fe00 (**OTA Firmware Update Service**)
  - **Characteristics**
    - 02f00000-0000-0000-0000-00000000ff00 (**READ**)
    - 02f00000-0000-0000-0000-00000000ff01 (**WRITE, WRITE NO RESPONSE**)
    - 02f00000-0000-0000-0000-00000000ff02 (**NOTIFY, READ**)
    - 02f00000-0000-0000-0000-00000000ff03 (**READ**)

All communication takes place by writing packets to characteristic `0783b03e-7735-b5a0-1760-a305d2795cb2` and receiving responses from the cube on characteristic `0783b03e-7735-b5a0-1760-a305d2795cb1`.

The cube advertises its own MAC address in the `Manufacturer Specific Data` of advertising packets with CIC `0x0000` if the cube is bound to an account in the WCU Cube app, and with CIC `0x0100` otherwise.

# Packet Structure

Packets sent to/received from the cube typically have a length of `20 bytes`. They consist of a `1 byte` **message type** and up to `19 bytes` of data (padded with trailing zeroes up to `19 bytes`). Assume big-endian unless otherwise stated.

# Encryption

The cube uses the same encryption scheme as **GAN Gen2 Smart Cubes** (AES-128-CBC using key and IV derived from a "root" key and IV + the cube's MAC address reversed as a "salt"). See **afedotov**'s implementation [here](https://github.com/afedotov/gan-web-bluetooth/blob/72567da984ad8605169a637674683a0f5ce920c5/src/gan-cube-encrypter.ts#L17) for reference.

Root key: `0x15773A5C670E2D1F17672A139B675257`\
Root IV: `0x11232625862A2C3B55067F317E672157`

# Message Types

## Cube Info (0xA1)

A request can be made for cube info by sending a packet with **message type** `0xA1` and no data:
`A100000000000000000000000000000000000000`

The cube responds with a packet in the following format:
| Bits | Type | Length | Description |
| :-------: | :---: | :---------------: | :------------------------: |
| 0 - 7 | u8 | 8 bits (1 byte) | Message Type (0xA1) |
| 8 - 71 | str | 64 bits (8 bytes) | Model Name (i.e. WCU_MY32) |
| 72 - 79 | u8 | 8 bits (1 byte) | Hardware Version (Major) |
| 80 - 87 | u8 | 8 bits (1 byte) | Hardware Version (Minor) |
| 88 - 95 | u8 | 8 bits (1 byte) | Software Version (Major) |
| 96 - 103 | u8 | 8 bits (1 byte) | Software Version (Minor) |
| 104 | bool | 1 bit | ? |
| 105 | bool | 1 bit | Gyro Enabled |
| 106 | bool | 1 bit | Gyro Functional? |
| 107 | bool | 1 bit | ? |
| 108 | bool | 1 bit | ? |
| 109 - 116 | u8 | 8 bits (1 byte) | Move Counter/Serial |
| 117 - 139 | ? | 22 bits | Unknown |

## Cube Status (Facelet State) (0xA3)

A request can be made for cube status (current facelet state) by sending a packet with **message type** `0xA3` and no data:
`A300000000000000000000000000000000000000`

The cube responds with a packet in the following format:
| Bits | Type | Length | Description |
| :-------: | :----------------------: | :-----------------: | :-----------------: |
| 0 - 7 | u8 | 8 bits (1 byte) | Message Type (0xA3) |
| 8 - 151 | Facelet Data (see below) | 144 bits (18 bytes) | Facelet Data |
| 152 - 159 | u8 | 8 bits (1 byte) | Move Counter/Serial |

Each facelet colour is encoded as a 3-bit unsigned integer. A total of 48 facelets are encoded for a total of `48 * 3 = 144` bits.

| Num | Colour |
| :-: | :----: |
|  0  | Green  |
|  1  |  Blue  |
|  2  | White  |
|  3  | Yellow |
|  4  | Orange |
|  5  |  Red   |

Each set of 24 bits (3 bytes) represents 8 facelets (i.e. a face without the centre colour). The face order is `FBUDLR`, in standard WCA scrambling orientation (green front, white top), meaning the facelet order is:

`FFFFFFFFBBBBBBBBUUUUUUUUDDDDDDDDLLLLLLLLRRRRRRRR`

A fully solved cube will, therefore, have the following facelet data (after parsing every 3 bits into an unsigned integer):

000000001111111122222222333333334444444455555555

## Cube Power (0xA4)

A request can be made for cube power (current battery level) by sending a packet with **message type** `0xA4` and no data:
`A400000000000000000000000000000000000000`

The cube responds with a packet in the following format:
| Bits | Type | Length | Description |
| :----: | :---: | :--------------: | :--------------------: |
| 0 - 7 | u8 | 8 bits (1 byte) | Message Type (0xA4) |
| 8 - 15 | u8 | 8 bits (1 bytes) | Battery Level (0-100%) |

## Cube Move (0xA5)

When a move is made on the cube, it sends a packet with **message type** `0xA5` in the following format:
| Bits | Type | Length | Description |
| :------: | :----: | :----------------: | :-----------------------------------------: |
| 0 - 7 | u8 | 8 bits (1 byte) | Message Type (0xA5) |
| 8 - 87 | u16[5] | 80 bits (10 bytes) | Last 5 move times (ms) |
| 88 - 95 | u8 | 8 bits (1 bytes) | Move Counter/Serial |
| 96 - 120 | u8[5] | 25 bits | Last 5 moves (5 bits each, see table below) |

Move table:

| Num | Move |
| :-: | :--: |
|  0  |  F   |
|  1  |  F'  |
|  2  |  B   |
|  3  |  B'  |
|  4  |  U   |
|  5  |  U'  |
|  6  |  D   |
|  7  |  D'  |
|  8  |  L   |
|  9  |  L'  |
| 10  |  R   |
| 11  |  R'  |

## Gyroscope (0xAB)

Gyroscope packets are constantly sent by the cube with **message type** 0xAB in the following format:
| Bits | Type | Length | Description |
| :-----: | :--------: | :-----------------: | :---------------------------------------: |
| 0 - 7 | u8 | 8 bits (1 byte) | Message Type (0xAB) |
| 8 - 135 | Quaternion | 128 bits (16 bytes) | Quaternion (see decoding procedure below) |

The quaternion is encoded as four `4 byte` signed integers in the order `(w, x, -z, y)` in **little-endian byte order**.\
Each chunk of 4 bytes can be converted to the correct floating-point representation as follows:

1. Let `a` be equal to the 4 byte chunk (i.e. an array of 4 bytes)
2. If the MSB of `a[0]` is set (i.e `(a[0] >> 7) & 1 == 1`), subtract `1` from `a[1]`
3. Parse `a` as a signed 32-bit integer in **little-endian byte order**, let `b` be equal to the resulting integer
4. If necessary in your language of choice, **cast** `b` from a 32-bit signed integer to a 32-bit float, then multiply `b` by `2^(-30)` (`9.3132257e-10`). The result will be the correct floating-point representation.

For example, given the packet `0xAB47882CFFF873493FA2B87509ECB43AFF000000`:

- 0xAB is the message type
- Our first 4-byte chunk is `0x47882CFF`
  - `(0x47 >> 7) & 1` == `0`, therefore we continue to the next step
  - We parse `0x47882CFF` as a 32-bit signed integer in **little-endian byte order**, the result is `-13858745`
  - We multiply `-13858745` by `2^(-30)`, giving `-0.01290696207433939`
  - Thus, `w = -0.01290696207433939`
- Our second 4-byte chunk is `0xF873493F`
  - `(0xF8 >> 7) & 1` == `1`, therefore we subtract `1` from `0x73` = `0x72`, giving `0xF872493F` as our new value
  - We parse `0xF872493F` as a 32-bit signed integer in **little-endian byte order**, the result is `1061778168`
  - We multiply `1061778168` by `2^(-30)`, giving `0.9888579770922661`
  - Thus, `x = 0.9888579770922661`
- Our third 4-byte chunk is `0xA2B87509`
  - `(0xA2 >> 7) & 1` == `1`, therefore we subtract `1` from `0xB8` = `0xB7`, giving `0xA2B77509` as our new value
  - We parse `0xA2B77509` as a 32-bit signed integer in **little-endian byte order**, the result is `158709666`
  - We multiply `158709666` by `2^(-30)`, giving `0.14780989475548267`
  - Thus, `-z = 0.14780989475548267`, and therefore, `z = -0.14780989475548267`
- Our fourth 4-byte chunk is `0xECB43AFF`
  - `(0xEC >> 7) & 1` == `1`, therefore we subtract `1` from `0xB4` = `0xB3`, giving `0xECB33AFF` as our new value
  - We parse `0xECB33AFF` as a 32-bit signed integer in **little-endian byte order**, the result is `-12930068`
  - We multiply `-12930068` by `2^(-30)`, giving `-0.012042064219713211`
  - Thus, `y = -0.012042064219713211`
- Our resulting `Quaternion(x, y, z, w)` is therefore `Quaternion(0.9888579770922661, -0.012042064219713211, -0.14780989475548267, -0.01290696207433939)`
