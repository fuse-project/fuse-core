# USB Over Local-Area Network Protocol (USB-LANP)

**Version:** 0.0.2

-----

## 1\. Protocol Overview

The USB-LANP is a decentralized, peer-to-peer protocol for sharing USB devices on a local network. It's built for zero-configuration, using UDP multicast for discovery and negotiation. The protocol is resilient and uses a lease-based system for device access. This version refines the broadcast-based negotiation model, making it more robust and providing explicit control over device release, while using a globally unique device identifier (GUID) for location independence.

-----

## 2\. Network and Transport Layer

  - **UDP Multicast:** Used on a dedicated port (`51101`) for all communication, including device discovery, state advertisement, negotiation, and control. Using a single multicast channel for all messages ensures that every participating device has a consistent and up-to-date view of the network state, which is crucial for conflict detection. Devices must join the multicast group `239.255.51.101`.
  - **USBIP**: Used to actually transfer the data from the USB port to the virtual USB driver

-----

## 3\. Message Format

All messages are plain text, terminated by a newline character (`\n`). Each message begins with a type identifier, followed by a series of fields separated by a pipe (`|`).

```
<MESSAGE_TYPE>|<FIELD_1>|<FIELD_2>|...|<FIELD_N>\n
```

-----

## 4\. Data Structures

  - **device\_id:** A globally unique identifier for each participating host device. Recommended to use the device's MAC address.
  - **usb\_guid:** A globally unique identifier for a USB device. If a device has a unique serial number, this is a combination of VID:PID:SERIAL. If not, the host generates a unique VID:PID:<HASH>, where <HASH> is a hash of VID:PID. 
  - **timestamp:** A UNIX timestamp representing a point in time (e.g., for lease expiration). All devices must have synchronized clocks for this to be effective.

-----

## 5\. Message Definitions

### 5.1. Discovery, Advertisement, and Negotiation (UDP Multicast)

All messages below are sent via UDP multicast. This ensures that every peer sees every negotiation step, enabling immediate conflict detection.

  - **ADVERTISEMENT:** Sent periodically (e.g., every 10 seconds) by all hosts to announce their presence and the status of their connected USB devices.

      - **Fields:**
          - `device_id` (sender's ID)
          - A comma-separated list of device states. Each device is defined as `<usb_guid>:<state>:<client_id>:<lease_end_timestamp>`.
      - **Example:**
        ```
        ADVERTISEMENT|00:1A:2B:3C:4D:5E|046D:C52B:12345678:CLAIMED:F1:B2:C3:D4:E5:F6:1672531200,8086:048A:ABCDEF:UNCLAIMED
        ```

  - **CLAIM\_REQUEST:** Sent by a client to request control of a specific USB device.

      - **Fields:**
          - `device_id` (sender's ID)
          - `usb_guid` (the device being requested)
          - `request_id` (a unique ID for this request)
          - `force` (a boolean flag: `TRUE` or `FALSE`)
      - **Example (normal request):**
        ```
        CLAIM_REQUEST|F1:B2:C3:D4:E5:F6|046D:C52B:12345678|REQ-12345|FALSE
        ```
      - **Example (forced request):**
        ```
        CLAIM_REQUEST|F1:B2:C3:D4:E5:F6|046D:C52B:12345678|REQ-12346|TRUE
        ```

  - **FREE:** Sent by any device to force a currently claimed USB device to become UNCLAIMED. This message is also useful for a device to "free" itself without a claim to a new client. The usb_guid in the message acts as the identifier for the device that should be released. The client with the corresponding claim must immediately release it.

      - **Fields:**
          - `usb_guid` (the USB device to be released)
      - **Example:**
        ```
        FREE|046D:C52B:12345678
        ```

  - **CLAIM\_GRANT:** Sent by the host that physically holds the USB device to grant a claim.

      - **Fields:**
          - `device_id` (host's ID)
          - `usb_guid`
          - `lease_end_timestamp`
          - `host_ip` (the ip address of the device that physically holds the USB device; required by USBIP to connect)
          - `host_busid` (the bus id of the port in which the device is plugged in; required by USBIP to connect)
          - `request_id` (matching the original `CLAIM_REQUEST`)
      - **Example (finite lease):**
        ```
        CLAIM_GRANT|00:1A:2B:3C:4D:5E|046D:C52B:12345678|1672531200|192.168.1.100|1-1.2|REQ-12345
        ```
      - **Example (indefinite lease):**
        ```
        CLAIM_GRANT|00:1A:2B:3C:4D:5E|046D:C52B:12345678|NULL|192.168.1.100|1-1.2|REQ-12346
        ```

  - **CLAIM\_DENIED:** Sent by the host to deny a claim request. 

      - **Fields:**
          - `device_id` (host's ID)
          - `usb_guid`
          - `reason` (`already_claimed`, `invalid_device`)
          - `request_id`
      - **Example:**
        ```
        CLAIM_DENIED|00:1A:2B:3C:4D:5E|046D:C52B:12345678|already_claimed|REQ-12345
        ```

  - **RELEASE\_CLAIM:** Sent by a client to voluntarily release a device. 

      - **Fields:**
          - `device_id` (sender's ID)
          - `usb_guid`
      - **Example:**
        ```
        RELEASE_CLAIM|F1:B2:C3:D4:E5:F6|046D:C52B:12345678
        ```

  - **DESIRED\_STATE:** Sent by a central contol UI (e.g. a Web UI Host) to all peers to declare a desired network state. 

      - **Fields:**
          - `device_id` (Web UI Host's ID)
          - A comma-separated list of desired device assignments. Each assignment is defined as `<usb_guid>:<target_client_id>`.
      - **Example:**
        ```
        DESIRED_STATE|1A:2B:3C:4D:5E:6F|046D:C52B:12345678:F1:B2:C3:D4:E5:F6,8086:048A:ABCDEF:B1:C2:D3:E4:F5:A6
        ```

-----

## 6\. Conflict Resolution and Negotiation Logic

The broadcast model is key to handling multiple clients simultaneously requesting the same device,
and it ensures a single source of truth for the network state.

### 6.1. Standard Negotiation

1.  A client wanting to use a USB device sends a **`CLAIM_REQUEST`** message with `force=FALSE`  via UDP multicast. 
2.  All devices on the network receive this request.
3.  The **host** that physically has the requested USB device checks its local state for that `usb_guid`.
4.  If the device is `UNCLAIMED`, the host immediately broadcasts a **`CLAIM_GRANT`** message. 
5.  The requesting client, upon receiving the `CLAIM_GRANT` with its matching `request_id`, knows it has successfully claimed the device and can initiate a USBIP connection to the host's IP address and the specified `host_busid`.
6.  All other devices on the network, seeing the `CLAIM_GRANT`, update their local state table to reflect that the device is now `CLAIMED` by the specified client.

### 6.2. Forced Claim

This process allows a new client to take over a device that is already claimed.

1. A client wanting to force using a USB device sends a **`CLAIM_REQUEST`** message with `force=TRUE` via UDP multicast. This message contains a unique `request_id`.
2. All devices on the network receive this request.
3. The **host** that physically has the requested USB device checks its local state for that `usb_guid`.
4. If the device is `CLAIMED` the host broadcasts a `FREE` message for the requested device.  The old client receiving the FREE message must immediately terminate its USB/IP connection and broadcast a RELEASE_CLAIM message. The RELEASE_CLAIM from the old client is a courtesy to update the network state faster, but the new claim takes precedence regardless.
5. The host broadcasts a **`CLAIM_GRANT`** message. 
6. The requesting client, upon receiving the `CLAIM_GRANT` with its matching `request_id`, knows it has successfully claimed the device and can initiate a USBIP connection to the host's IP address and the specified `tcp_port`.
7. All other devices on the network, seeing the `CLAIM_GRANT`, update their local state table to reflect that the device is now `CLAIMED` by the specified client.

### 6.3. Conflict Resolution: Multiple Claim Requests

A conflict occurs when two or more clients simultaneously send a **`CLAIM_REQUEST`** for the same `usb_guid`.

1.  **Simultaneous Requests:** Client A sends `CLAIM_REQUEST|A|GUID|REQ-A` and Client B sends `CLAIM_REQUEST|B|GUID|REQ-B`.
2.  **Host Response:** The host that physically holds the device receives both requests. The host's logic is to **grant the first valid request it processes**. Forced requests always get priority.
3.  **Conflict Detection:** Client B, which also sent a request for the same device, receives a `CLAIM_GRANT` from the host for Client A's request (`REQ-A`). Client B immediately recognizes that its own request (`REQ-B`) was not granted.
4.  **Backoff and Retry:** Client B will not proceed with its claim. Instead, it enters a random exponential backoff period before sending another `CLAIM_REQUEST` for the same `usb_guid`. This prevents a "thundering herd" problem and ensures a single client eventually wins the negotiation. The random backoff period ensures that retries don't collide.

### 6.4. Mismatch Resolution: Physical vs. Advertised State

A mismatch occurs when the network's advertised state (what's in the `ADVERTISEMENT` messages) doesn't match the reality of a host's USBIP connections.

1.  **Detection:** All hosts continuously listen to the `ADVERTISEMENT` messages from all other hosts to maintain a global state table. A host with a claimed device (`CLAIMED` state in its table) will also monitor its local USBIP connection. If the connection fails or is manually disconnected, the host's local state will be `UNCLAIMED`, but the network's advertised state will remain `CLAIMED`.
2.  **Resolution:** The host that originally granted the claim (and whose local connection is now broken) will stop advertising the device as `CLAIMED`. It will instead advertise the device as `UNCLAIMED` in its next periodic `ADVERTISEMENT` message.
3.  **Lease Expiration:** The claiming client must periodically renew its lease by sending a new `CLAIM_REQUEST`. If the lease expires because the client is disconnected or fails to renew, the host automatically moves the device's state to `UNCLAIMED` in its local table and its `ADVERTISEMENT` messages, making it available for others to claim. This self-healing mechanism handles graceful disconnections.

### 6.5. Indefinite Lease (`lease_end_timestamp = NULL`)

The `lease_end_timestamp` field in the `CLAIM_GRANT` message defines the expiration of a claim.

  - **Finite Lease:** If the value is a valid UNIX timestamp, the claim is valid until that time. The client must send a new `CLAIM_REQUEST` before the lease expires to renew it.
  - **Indefinite Lease:** If the value is `NULL`, the claim is permanent. This type of claim will **not** expire and must be voluntarily released by the client using a `RELEASE_CLAIM` message, or forcibly taken over by another client with a `CLAIM_REQUEST` that has the `force` flag set to `TRUE`, or overridden by a `DESIRED_STATE` message from the Web UI.

### 6.6. Web UI-Driven Conflict Resolution

The `DESIRED_STATE` message acts as a centralized command that all devices must obey.

1.  **Broadcast Command:** The Web UI Host broadcasts a `DESIRED_STATE` message. All devices receive this message.
2.  **Coordinated Release:** If a device is currently claimed by a client different from the one specified in the `DESIRED_STATE` message, the current client will immediately and voluntarily broadcast a **`RELEASE_CLAIM`** message. This action is taken because the `DESIRED_STATE` message is considered a higher-priority command.
3.  **New Claim Request:** The target client specified in the `DESIRED_STATE` message will then immediately broadcast a `CLAIM_REQUEST` for the device.

-----

## 7\. Plugin Interface Protocol

The plugin interface is also updated to be device-centric, mirroring the core protocol changes. It uses a local communication channel (e.g., Unix domain socket) for security and performance.

### 7.1. Message Format

  - `<MESSAGE_TYPE>|<REQUEST_ID>|<PAYLOAD>`

### 7.2. Core-to-Plugin Messages

  - **EVENT:**
      - `EVENT|<REQUEST_ID>|<EVENT_TYPE>,<DATA>`
      - Example: `EVENT|42|DEVICE_CLAIMED,046D:C52B:12345678,F1:B2:C3:D4:E5:F6`
  - **RESPONSE:**
      - `RESPONSE|<REQUEST_ID>|<STATUS>,<DATA>`
      - Example: `RESPONSE|43|OK,Device claimed`

### 7.3. Plugin-to-Core Messages

  - **QUERY:**
      - `QUERY|<REQUEST_ID>|<QUERY_TYPE>,<PARAMS>`
      - Example: `QUERY|44|DEVICE_STATUS,046D:C52B:12345678`
  - **COMMAND:**
      - `COMMAND|<REQUEST_ID>|<COMMAND_TYPE>,<PARAMS>`
      - Example: `COMMAND|45|RELEASE_CLAIM,046D:C52B:12345678`

### 7.4. Message Types and Semantics

  - **EVENT\_TYPE** examples: `DEVICE_CLAIMED`, `DEVICE_RELEASED`, `LEASE_EXPIRED`, `DEVICE_DISCOVERED`.
  - **QUERY\_TYPE** examples: `DEVICE_STATUS`, `DEVICE_LIST`, `LEASE_INFO`.
  - **COMMAND\_TYPE** examples: `CLAIM_DEVICE`, `RELEASE_CLAIM`, `RENEW_LEASE`.

-----