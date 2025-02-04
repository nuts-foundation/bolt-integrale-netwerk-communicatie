# Federated matrix.org server setup

This repository contains a config for setting up 3 federated matrix.org home servers.

There are multiple server implementations for the matrix protocol. The most popular one is [synapse](https://element-hq.github.io/synapse/latest/). This setup uses synapse.

Its purpose is to provide a simple way to domonstrate and test federation between matrix.org servers.

## Architecture

### Homeservers

The setup consists of 3 containers with a [synapse](https://element-hq.github.io/synapse/latest/) homeserver:

- matrix_apotheek
- matrix_huisarts
- matrix_vvt

These servers run in docker containers and expose their federation and client port on `8008`.
They accept federation from other servers based on a Certificate Authority.

The name of the homeserver is the same as the domain but with `matrix_` prepended.

### Reverse proxy

The servers are exposed to the host and each other through a reverse proxy. This reverse proxy is also a docker container and listens on ssl port `8448`. It forwards requests to the correct homeserver based on the domain name.

It also performs SSL termination and forwards the request to the homeserver over plain HTTP.

### Certificate Authority

The homeservers and reverse proxy use a self-signed certificate authority to sign their certificates. This CA is shared between the servers. It can be found in the `data/tls` directory.

### Clients

Currently only the vvt organisation has a client, which is an element client. It runs in `element_vvt` and is exposed to the host on port `8003`.

## Setup

To run the setup you need to have docker and docker-compose installed. Just run `docker-compose up` in the root of the repository.

### Host configuration

If you want to connect to the homeservers from the host you'll need map their domain names to your localhost. You can do this by adding the following lines to your `/etc/hosts` file:

```
127.0.0.1 apotheek
127.0.0.1 huisarts
127.0.0.1 vvt
```

### Admin user registration

The first time you need to add an admin user to the homeservers. You can do this by running the following command for each homeserver:

```
docker compose exec apotheek register_new_matrix_user http://apotheek:8008 -c /data/homeserver.yaml
```

### Homeserver configuration

The homeservers are configured in the `data/${env}/homeserver.yaml` file. You can change the configuration of the homeservers by editing this file and restarting the homeservers.

#### Generating a basic configuration

The synapse executable can generate a basic configuration file for you. You can do this by running the following command for each homeserver:

```bash
docker run -it --rm \
    --mount type=bind,src=./data/huisarts,dst=/data \
    -e SYNAPSE_SERVER_NAME=huisarts \
    -e SYNAPSE_REPORT_STATS=no \
    matrixdotorg/synapse:latest generate
```

This will generate a basic configuration file in the `data/huisarts/homeserver.yaml` file.

#### Configuration options

Notable configuration options are:

- `server_name`: The domain name of the homeserver
- `federation_custom_ca_list`: The path to the CA certificate authority which defines the trusted CAs for federation.
- `registration_shared_secret`: The shared secret for registering new users on the homeserver.
- `ip_range_whitelist`: The IP range that the homeserver can connect to. This is set to the ranges of docker networks. We might nog need this option, but it's there for now.

## Application Services

One of the nice features or goals of matrix.org is creating interoperability between different communication services. This is done through [application services](https://spec.matrix.org/latest/application-service-api/). These are services that connect to the homeserver and can send and receive messages on behalf of users.

### Configuration

The application service needs to be listed in the homeserver configuration file. You can do this by adding the following lines to the `homeserver.yaml` file:

See the [synapse AS documentation](https://element-hq.github.io/synapse/latest/application_services.html)

```yaml
app_service_config_files:
  - /data/my_as_config.yaml
```

with the follow example fonfiguration for a FHIR Bridge:

```yaml
id: "FHIR Bridge"
# url: "http://127.0.0.1:1234"
url: "http://host.docker.internal:1234"
as_token: "30c05ae90a248a4188e620216fa72e349803310ec83e2a77b34fe90be6081f46"
hs_token: "312df522183efd404ec1cd22d2ffa4bbc76a8c1ccf541dd692eef281356bb74e"
sender_localpart: "_fhir" # Will result in @_fhir:example.org
namespaces:
  users:
    - exclusive: true
      regex: "@_fhir_bridge_.*"
  aliases:
    - exclusive: false
      regex: "#_fhir_bridge_.*"
  rooms:
    - exclusive: false
      regex: "!*"
```

The config describes a `FHIR Bridge` which runs on the host machine on port `1234`. It has a shared secret token for receiving request from the home server (HS) and a shared secret token for making request to the HS.

Namespaces are used to define which users, aliases and rooms the application service can interact with. In this case the application service can interact with users with the localpart `_fhir_bridge_`, aliases with the localpart `_fhir_bridge_` and any room.

See more about namespaces in the [matric specificaton](https://spec.matrix.org/unstable/application-service-api/#registration).

## Implementing an Application Service

### Events

Matrix.org synchronizes state between home servers using events. See the [specification about events](https://spec.matrix.org/unstable/client-server-api/#events). A basic event has the following structure:

```json
{
  "age": 34,
  "content": {
    "body": "Hello, World!",
    "msgtype": "m.text"
  },
  "event_id": "$2tEZAQr2aA95IKiDeef213SZGfsn4cQ-vpBTGFJihmQ",
  "origin_server_ts": 1738683206178,
  "room_id": "!TtZEMmHFbejinwYnbQ:vvt",
  "sender": "@admin:vvt",
  "type": "m.room.message",
  "unsigned": {
    "age": 34
  },
  "user_id": "@admin:vvt"
}
```

### Application Service API

In order to interact with the homeserver, the application service needs to implement the [Application Service API](https://spec.matrix.org/unstable/application-service-api/). This API is used to receive events from the homeserver and to send events to the homeserver.

The Application Service API is an extension of the [Client-Server API](https://spec.matrix.org/unstable/client-server-api/).

#### Authentication

In order for the AS to make request to the HS, it needs to authenticate itself with the HS. This is done by adding the `Authorization` header to the request with the value `Bearer <hs_token>` as configured in the AS configiguration.

When the AS wants to perform an action on behalf of a user, it needs obtain an access token for that user by logging in. This can be done by sending a `POST` request to the `/_matrix/client/v3/login` endpoint with a body like:

```json
{
  "type": "m.login.application_service",
  "identifier": {
    "type": "m.id.user",
    "user": "@_fhir"
  }
}
```

See for more information [the specification about Server Admin Style permissions](https://spec.matrix.org/unstable/application-service-api/#server-admin-style-permissions)

#### Performing actions

A simple integraton should implement a few basic operations like:

- Create a room
- Invite a user to a room
- Accept an invitation to a room
- Send a message to a room

##### Creating a room

[See the specification about creating a room](https://spec.matrix.org/unstable/client-server-api/#creation).

To create a room, the AS needs to send a `POST` request to the `POST /_matrix/client/v3/createRoom` endpoint with a body like:

```json
{
  "preset": "public_chat",
  "room_alias_name": "my_room",
  "name": "My Room",
  "topic": "A room for discussing things"
}
```

response:

```json
{
  "room_id": "!sefiuhWgwghwWgh:example.com"
}
```

The above request creates a public room which can be joined by anyone.

##### Inviting a user to a room

[See the specification about inviting a user to a room](https://spec.matrix.org/unstable/client-server-api/#post_matrixclientv3roomsroomidinvite).

To invite a user to a room, the AS needs to send a `POST` request to the `/_matrix/client/v3/rooms/{roomId}/invite` endpoint with a body like:

```json
{
  "user_id": "@user:example.com"
}
```

##### Accepting an invitation to a room

After a user has been invited it can join by sending a `POST` reqeust to the `/_matrix/client/v3/join/{roomIdOrAlias}` endpoint with a body like:

```json
{
  "room_id": "!sefiuhWgwghwWgh:example.com"
}
```

##### Sending a message to a room

Matrix is a protocol for state synchronization bewtween home servers. Its use is not limited to chat applications. Therefore the protocol is designed to be extensible. Extensions are called [modules](https://spec.matrix.org/unstable/client-server-api/#modules). Instant messageging is one of the modules and described in [module 10.2](https://spec.matrix.org/unstable/client-server-api/#instant-messaging).

To send a text message to a room, the even type should be `m.room.message` and the content should contain a `msgtype` of `m.text` and a `body`:

```json
{
  "content": {
    "body": "This is an example text message",
    "msgtype": "m.text"
  },
  "event_id": "$143273582443PhrSn:example.org",
  "origin_server_ts": 1432735824653,
  "room_id": "!jEsUZKDJdhlrceRyVU:example.org",
  "sender": "@example:example.org",
  "type": "m.room.message",
  "unsigned": {
    "age": 1234,
    "membership": "join"
  }
}
```
