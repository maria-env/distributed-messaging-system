# Distributed Messaging System

A client–server instant-messaging system built for a Distributed Systems course. Users register, connect and exchange text messages or file attachments. It combines several distributed-systems technologies in one project:

- **TCP sockets** for the client ↔ server protocol (C server, Python client)
- **POSIX threads** for concurrent request handling
- **Sun RPC (ONC RPC)** for a remote operation-logging service
- **SOAP web service** (Python) that normalises message text before it is sent

> Distributed Systems course project (Universidad Carlos III de Madrid, 2025–26).

## Architecture

```
                    SOAP (HTTP)
   ┌──────────┐  normalize(msg)   ┌─────────────────┐
   │ client.py│ ◄───────────────► │ web_service.py  │
   └────┬─────┘                   └─────────────────┘
        │ TCP                                    
        ▼                                        Sun RPC
   ┌──────────┐ ───────────────────────────► ┌────────────┐
   │ server.c │      log_operation()         │ log_server │
   └──────────┘                              └────────────┘
        ▲
        │ TCP (server → client listener, for message delivery and ACKs)
        ▼
   other clients ◄── TCP, peer-to-peer (GETFILE) ──► clients
```

- **Server (`server.c`, `db.c`)** – accepts TCP connections and serves each one in a detached thread. A semaphore caps the number of concurrent threads (`MAX_HILOS = 5`) and a mutex protects the in-memory user/message database. Users and pending messages are persisted to `db_users.txt` and `db_messages.txt`.
- **Client (`client.py`)** – interactive shell. After `CONNECT`, it starts a listener thread on a random free port so the server can push incoming messages and delivery acknowledgements, and so peers can request files directly.
- **Web service (`web_service.py`)** – SOAP 1.1 service (Spyne) exposing `normalize`, which collapses repeated whitespace. If it is unreachable, the client prints a warning and sends the original text.
- **Log service (`log.x`, `log_server.c`)** – RPC interface `log_operation(user, operation, filename)`. The server calls it on every operation, and the log server prints each entry.

## Features

| Command | Description |
|---------|-------------|
| `REGISTER <user>` | Create a user |
| `UNREGISTER <user>` | Delete a user |
| `CONNECT <user>` | Go online. Messages stored while offline are delivered immediately |
| `DISCONNECT <user>` | Go offline |
| `USERS` | List connected users with their IP and port |
| `SEND <user> <message>` | Send a message. If the recipient is offline, it is stored until they connect |
| `SENDATTACH <user> <file> <message>` | Send a message that references a file |
| `GETFILE <user> <remote_file> <local_file>` | Download a file directly from another connected client |
| `QUIT` | Close the client |

Each message gets an incrementing ID per sender, and the sender receives an acknowledgement once it is delivered.

## Project structure

| File | Description |
|------|-------------|
| `server.c`, `server.h` | Messaging server: sockets, threads, request handlers, RPC logging calls |
| `db.c`, `db.h` | In-memory database (linked lists) with file persistence |
| `client.py` | Interactive client and listener thread |
| `web_service.py` | SOAP message-normalisation service (port 8000) |
| `log.x` | RPC interface definition (`rpcgen` generates the stubs) |
| `log_server.c` | RPC log server implementation |
| `log_client.c` | `rpcgen` client template (not built by the Makefile) |
| `Makefile` | Builds `server` and `log_server` |
| `prep_server.sh`, `prep_client.sh` | Helper scripts to start the server and the client |

## Requirements

- Linux with `gcc`, `make`, `rpcgen` and `libtirpc` (e.g. `apt-get install build-essential libtirpc-dev rpcsvc-proto`)
- Python 3.8+ with `spyne`, `zeep` and `lxml`

```bash
pip3 install spyne lxml zeep
```

## Build

```bash
make
```

This runs `rpcgen` on `log.x` and builds the `server` and `log_server` executables. Use `make clean` to remove generated files.

## Run

Start the components in this order. They can run on different machines or containers.

**1. RPC log server**

```bash
./log_server
```

**2. Messaging server**

```bash
export LOG_RPC_IP=<log_server_address>
./server <port>
# or: ./prep_server.sh <log_server_address> <port>
```

If `LOG_RPC_IP` is not set, the server runs without logging. Run it from the project directory, because the database files are read from the current directory.

**3. Web service and client**

```bash
python3 web_service.py &
python3 client.py -s <server_address> -p <port>
```

`prep_client.sh <server_address> <port>` also installs the Python dependencies (using `apt-get` and `pip3`), starts the web service in the background, and launches the client.

The client port must be in the range 1024–65535.

### Example session

```
c> REGISTER alice
c> REGISTER bob
c> CONNECT alice
c> USERS
c> SEND bob Hello   Bob
c> QUIT
```

The server console prints lines such as `s> REGISTER alice OK`. The web service turns `Hello   Bob` into `Hello Bob`.

## Notes and limitations

- The web service URL is hardcoded to `http://localhost:8000/?wsdl`, so each client machine must run its own copy.
- Strings are limited to 255 bytes, and `GETFILE` does not use any authentication.
- User names cannot contain spaces.
- The database is stored in plain-text files and kept entirely in memory.

## Authors

Team project by María Arias Rodríguez and Jorge Ignacio Castañeda Vallenilla.
