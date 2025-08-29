# USB Over Local-Area Network Protocol (USB-LANP)

**Version:** 0.0.1

---

## 1. Protocol Overview

USB-LANP is a decentralized, peer-to-peer protocol for sharing USB devices on a local network. It operates on a zero-configuration model using UDP multicast for device discovery and TCP for state synchronization and negotiation. The protocol is designed to be resilient, with a self-healing lease-based system for port access.

---

## 2. Network and Transport Layer

- **UDP Multicast:** Used on a dedicated port (`51101`) for device discovery and state advertisement. Devices should join the multicast group `239.255.51.101`.
- **TCP:** Used for all point-to-point communication, including negotiation and control. TCP provides reliable, ordered, and error-checked delivery of messages.

---

## 3. Message Format

All messages are plain text, terminated by a newline character (`\n`). Each message begins with a type identifier, followed by a series of fields separated by a pipe (`|`).

```
<MESSAGE_TYPE>|<FIELD_1>|<FIELD_2>|...|<FIELD_N>\n
```

---

## 4. Data Structures

- **device_id:** A globally unique identifier for each participating device. It is recommended to use the device's MAC address, as it's non-configurable and permanent.
- **bus_id:** The string identifier for a USB port (e.g., `1-1.2`) as provided by the `lsusb` command on a Linux system.
- **timestamp:** A UNIX timestamp representing a point in time (e.g., for lease expiration). All devices must have synchronized clocks for this to be effective.

---

## 5. Message Definitions

### 5.1. Discovery and Advertisement (UDP)

**ADVERTISEMENT:** Sent periodically (e.g., every 10 seconds) by all devices to announce their presence and the status of their USB ports.

- **Fields:**
  - `device_id` (sender's ID)
  - A comma-separated list of port states. Each port is defined as `<bus_id>:<state>:<client_id>:<lease_end_timestamp>`.
- **Example:**

```
ADVERTISEMENT|00:1A:2B:3C:4D:5E|1-1.2:CLAIMED:F1:B2:C3:D4:E5:F6:1672531200,1-1.3:UNCLAIMED
```

### 5.2. Negotiation and Control (TCP)

- **CLAIM_REQUEST:** Sent by a client to a host to request control of a specific port. This message serves as a request and a lease renewal.
  - **Fields:**
    - `device_id` (sender's ID)
    - `bus_id` (the port being requested)
  - **Example:**

    ```
    CLAIM_REQUEST|F1:B2:C3:D4:E5:F6|1-1.2
    ```

- **CLAIM_GRANT:** Sent by a host to a client to grant a claim.
  - **Fields:**
    - `device_id` (host's ID)
    - `bus_id`
    - `lease_end_timestamp` (the time the claim expires)
  - **Example:**

    ```
    CLAIM_GRANT|00:1A:2B:3C:4D:5E|1-1.2|1672531200
    ```

- **CLAIM_DENIED:** Sent by a host to deny a claim request.
  - **Fields:**
    - `device_id` (host's ID)
    - `bus_id`
    - `reason` (e.g., `already_claimed`, `invalid_port`)
  - **Example:**

    ```
    CLAIM_DENIED|00:1A:2B:3C:4D:5E|1-1.2|already_claimed
    ```

- **RELEASE_CLAIM:** Sent by a client to voluntarily release a port.
  - **Fields:**
    - `device_id` (sender's ID)
    - `bus_id`
  - **Example:**

    ```
    RELEASE_CLAIM|F1:B2:C3:D4:E5:F6|1-1.2
    ```

### 5.3. Web UI State Management (TCP)

- **DESIRED_STATE:** Sent by the Web UI Host to all peers to declare a desired network state. This is a broadcast message.
  - **Fields:**
    - `device_id` (Web UI Host's ID)
    - A comma-separated list of desired port assignments. Each assignment is defined as `<bus_id>:<target_client_id>`.
  - **Example:**

    ```
    DESIRED_STATE|1A:2B:3C:4D:5E:6F|1-1.2:F1:B2:C3:D4:E5:F6,1-1.3:B1:C2:D3:E4:F5:A6
    ```

---

## 6. State Machine and Logic

Each device maintains a local state for every known USB port on the network.

- **UNCLAIMED:** The port is not in use.
- **CLAIMED:** The port is in use by a remote device.
- **LOCAL:** The port is physically connected to and in use by the local host device.

### Core Logic

- **Discovery:** Devices listen for `ADVERTISEMENT` messages to build a comprehensive state table of the network.
- **Negotiation:**
  - **Client:** A client needing a port sends a `CLAIM_REQUEST` to the host device. It will retry with a random backoff if denied.
  - **Host:** A host receiving a `CLAIM_REQUEST` grants it if the port is `UNCLAIMED` and starts a lease. It denies if the port is already `CLAIMED` or `LOCAL`.
- **Lease Management:** A `CLAIMED` port is only valid until its `lease_end_timestamp`. The claiming client must send a `CLAIM_REQUEST` again before the lease expires to renew it. If a lease expires, the host moves the port back to `UNCLAIMED`. This handles disconnected clients gracefully.

---

## 7. Web UI Integration

The Web UI Host is a standard peer that observes all network traffic.

- **State Observation:** The Web UI Host listens to all `ADVERTISEMENT` messages to build a real-time view of the network.
- **Configuration:** A user makes changes in the web interface, defining the desired state.
- **Protocol Injection:** The Web UI Host translates the desired state into `DESIRED_STATE` messages and broadcasts them.
- **Decentralized Negotiation:** Upon receiving a `DESIRED_STATE` message, all devices enter a negotiation phase:
  - If a port is currently `CLAIMED` by a client other than the one specified in the `DESIRED_STATE` message, the current client will send a `RELEASE_CLAIM` message.
  - The target client specified in the `DESIRED_STATE` message will then immediately send a `CLAIM_REQUEST` for the port.
  - This triggers the standard claim negotiation, with the desired configuration taking precedence due to all devices acting on the same command simultaneously.

---

## 8. Plugin Interface Protocol

To enable extensibility, each device supports a local plugin interface that allows external scripts ("plugins") to interact with the core service. This interface is designed to be cross-platform and implementation-agnostic, ensuring compatibility across different environments.

### 8.1. Communication Channel

- The core service exposes a local communication channel accessible only from the same device (e.g., via a local TCP socket, Unix domain socket, or named pipe).
- The channel is bidirectional and supports concurrent requests and responses.
- All messages are encoded as UTF-8 text, with each message terminated by a newline character (`\n`).

### 8.2. Message Format

- Each message is a single line of text, structured as a pipe-separated (`|`) list of fields:
  - `<MESSAGE_TYPE>|<REQUEST_ID>|<PAYLOAD>`
- `MESSAGE_TYPE`: The type of message (see below).
- `REQUEST_ID`: A unique identifier for correlating requests and responses.
- `PAYLOAD`: The message-specific data, further structured as needed (e.g., pipe- or comma-separated fields).

### 8.3. Core-to-Plugin Messages

- **EVENT:** Notifies the plugin of a state change or event in the core.
  - `EVENT|<REQUEST_ID>|<EVENT_TYPE>,<DATA>`
  - Example: `EVENT|42|PORT_CLAIMED,1-1.2,F1:B2:C3:D4:E5:F6`
- **RESPONSE:** Returns the result of a plugin-initiated request.
  - `RESPONSE|<REQUEST_ID>|<STATUS>,<DATA>`
  - Example: `RESPONSE|43|OK,Port released`

### 8.4. Plugin-to-Core Messages

- **QUERY:** Requests information from the core.
  - `QUERY|<REQUEST_ID>|<QUERY_TYPE>,<PARAMS>`
  - Example: `QUERY|44|PORT_STATUS,1-1.2`
- **COMMAND:** Instructs the core to perform an action.
  - `COMMAND|<REQUEST_ID>|<COMMAND_TYPE>,<PARAMS>`
  - Example: `COMMAND|45|RELEASE_CLAIM,1-1.2`

### 8.5. Message Types and Semantics

- **EVENT_TYPE** examples: `PORT_CLAIMED`, `PORT_RELEASED`, `LEASE_EXPIRED`, `DEVICE_DISCOVERED`, etc.
- **QUERY_TYPE** examples: `PORT_STATUS`, `DEVICE_LIST`, `LEASE_INFO`, etc.
- **COMMAND_TYPE** examples: `CLAIM_PORT`, `RELEASE_CLAIM`, `RENEW_LEASE`, etc.
- All types and parameters must be documented and versioned to ensure forward compatibility.

### 8.6. Error Handling

- If a message is malformed or an operation fails, the core responds with a `RESPONSE` message containing an error status and description.
  - Example: `RESPONSE|46|ERROR,Invalid port ID`

### 8.7. Security and Isolation

- The plugin interface is only accessible locally and is not exposed to the network.
- Plugins are isolated from each other and have no direct access to the core's internal state beyond the defined protocol.

---
