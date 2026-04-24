# Distributed File System

A distributed file system (DFS) simulation built in Python using TCP sockets. Files are spread across multiple servers and clients interact with the system through a single unified interface — no matter which server holds the file, the client always gets transparent access to it.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Requirements](#requirements)
- [Project Structure](#project-structure)
- [Setup & Configuration](#setup--configuration)
- [Running the System](#running-the-system)
- [Client Commands](#client-commands)
- [How It Works](#how-it-works)

---

## Overview

This project simulates a distributed file system across a minimum of three machines (or separate processes on the same machine). Clients connect to any server and can transparently create, read, write, append to, delete, and list files stored anywhere in the cluster. File replication is handled automatically in the background.

---

## Architecture

```
  Client A          Client B
     |                 |
     +--------+--------+
              |
     +-----------------+
     |                 |                 |
  Server A  <-->  Server B  <-->  Server C
  (Root/)         (Root/)          (Root/)
```

- **Servers** communicate with each other over inter-server ports. In a multi-machine deployment, `5555` is an example default per host; when running multiple server processes on one machine, each server instance must use a **unique** inter-server port (for example, `5555`, `5556`, `5557`).
- **Clients** connect to a server's client-facing port. In a multi-machine deployment, `9999` is an example default per host; when running multiple server processes on one machine, each server instance must use a **unique** client port (for example, `9999`, `9998`, `9997`).
- Each server maintains a **global file list** — a registry of every file in the cluster and which server owns it.
- If a requested file lives on a different server, the client is transparently **redirected** to the correct server.

---

## Features

- **Transparent file access** — clients see a single unified namespace regardless of where files are physically stored.
- **Automatic replication** — when a file is created, it is replicated to one other server for redundancy.
- **Replication sync** — writes and appends propagate updates to all replicas automatically.
- **Concurrent connections** — each client and server connection is handled in its own thread with mutex locking for thread safety.
- **File operations** — create, read, write (overwrite), append, delete, and list.
- **Supported file types** — `.txt`, `.c`, and `.py` files.

---

## Requirements

- Python 3.x
- [`leafpad`](https://en.wikipedia.org/wiki/Leafpad) text editor (used to open files for editing in the GUI; Linux only)
  - Install with: `sudo apt install leafpad`

---

## Project Structure

```
Distributed-File-System/
├── server.py          # Main server script (shared base)
├── client.py          # Main client script
├── Server_A/
│   ├── server.py      # Server A instance (pre-configured IPs)
│   └── Root/          # Server A file storage directory
├── Server_B/
│   ├── server.py      # Server B instance
│   └── Root/
├── Server_C/
│   ├── server.py      # Server C instance
│   └── Root/
├── Client_A/
│   ├── client.py      # Client A instance
│   └── Root/          # Local cache directory for Client A
└── Client_B/
    ├── client.py
    └── Root/
```

---

## Setup & Configuration

### Server Configuration

Open `server.py` (or the relevant `Server_X/server.py`) and update the following variables:

```python
# Your machine's IP address
my_host = "192.168.1.100"

# IPs of the other two servers in the cluster
servers_to_connect = [['192.168.1.101', 5555], ['192.168.1.102', 5555]]

# Port for inter-server communication (default: 5555)
port = 5555

# Port for client connections (default: 9999)
c_port = 9999

# Local file storage directory
root = "Root/"
```

### Client Configuration

The client does not require pre-configuration. Server IP and port are entered at runtime.

Optionally, update the `root` variable in `client.py` to change the local cache directory:

```python
root = "Root/"
```

---

## Running the System

Each server and client must be run from its own directory so that their `Root/` folders are kept separate.

### 1. Start each server (on separate machines or terminal windows)

```bash
cd Server_A
python3 server.py
```

```bash
cd Server_B
python3 server.py
```

```bash
cd Server_C
python3 server.py
```

Servers will automatically attempt to connect to each other using the IPs in `servers_to_connect` and exchange their local file lists.

### 2. Start a client

```bash
cd Client_A
python3 client.py
```

When prompted, enter the IP address and port of any running server:

```
Enter server IP
192.168.1.100
Enter server port
9999
```

> **Note:** If running everything on one machine, use `127.0.0.1` (localhost) as the IP and make sure each server/client runs in its own directory.

---

## Client Commands

Once connected, the client accepts the following commands:

| Command              | Description                                              |
|----------------------|----------------------------------------------------------|
| `list`               | List all files available across the entire cluster       |
| `create <filename>`  | Create a new empty file (e.g., `create notes.txt`)       |
| `read <filename>`    | Read a file (view in terminal or open in `leafpad`)      |
| `write <filename>`   | Overwrite a file's contents (opens in `leafpad`)         |
| `append <filename>`  | Append content to a file (opens in `leafpad`)            |
| `delete <filename>`  | Delete a file from the cluster                           |
| `exit`               | Disconnect from the server and quit                      |

**Supported file extensions:** `.txt`, `.c`, `.py`

**Example session:**

```
>>> create hello.txt
file created on server

>>> list
hello.txt

>>> write hello.txt
(leafpad opens — edit the file and save, then close leafpad)

>>> read hello.txt
Press 1 to read in terminal
Press 2 to read in editor
1
Hello, Distributed File System!

>>> delete hello.txt
file deleted
```

---

## How It Works

1. **Server startup** — each server loads its local `Root/` directory into its file list, then connects to the other servers and exchanges file lists to build a global view of all files in the cluster.

2. **File creation** — when a client creates a file, the server stores it locally, broadcasts the new entry to all connected servers (so they update their global lists), then replicates the file to one other server for redundancy.

3. **File access (read/write/append/delete)** — the server looks up the file in the global list. If the file is stored locally, it serves the request directly. If it's on another server, it sends the client a redirect message (`connect to <ip>`) and the client reconnects to the correct server automatically.

4. **Replication sync** — after every write or append, the updated file is pushed to all servers that hold a replica.

5. **Concurrency** — each client and server connection runs in a separate thread. A `threading.Lock` guards all shared data structures (global file list, connected server list) to prevent race conditions.
