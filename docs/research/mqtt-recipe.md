# MQTT broker recipe for two remote pets

Research ticket: issue 5, "Find the Mosquitto broker recipe for two remote pets".
Resolved 2026-11-02. Desk research only: no server was deployed, no SSH access was used.
All claims below are settled from primary sources, linked inline. Where a fact could
not be established from a primary source, the doc says so explicitly (section 6).

Primary sources used:

- [mosquitto.conf(5) man page](https://mosquitto.org/man/mosquitto-conf-5.html) (canonical directive reference)
- [mosquitto-tls(7) man page](https://mosquitto.org/man/mosquitto-tls-7.html) (self-signed CA/cert generation)
- [mosquitto_passwd(1) man page](https://mosquitto.org/man/mosquitto_passwd-1.html) (password file creation)
- [eclipse-mosquitto Docker image docs](https://mosquitto.org/documentation/) and repo [docker/ dir](https://github.com/eclipse-mosquitto/mosquitto/tree/master/docker) (mount points, default conf, uid/gid 1883)
- [Docker Hub eclipse-mosquitto tags](https://hub.docker.com/_/eclipse-mosquitto) (available tags: 2.0.22, 2.1.2, latest)
- [MQTT 3.1.1 spec (OASIS)](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html) (clean session, QoS 1, retained, will, keepalive, ordering)
- [micropython-lib umqtt.simple](https://github.com/micropython/micropython-lib/tree/master/micropython/umqtt.simple) (source + README + examples)
- [micropython-lib umqtt.robust](https://github.com/micropython/micropython-lib/tree/master/micropython/umqtt.robust) (source and README)
- [MicroPython ssl module docs](https://docs.micropython.org/en/latest/library/ssl.html) (CERT_REQUIRED, cadata, mbedTLS notes)
- [MicroPython ESP32 frozen modules manifest](https://github.com/micropython/micropython/blob/master/ports/esp32/boards/manifest.py) (umqtt.simple and umqtt.robust are frozen into ESP32 firmware)
- [Let's Encrypt DNS-01 challenge docs](https://letsencrypt.org/docs/challenge-types/) (DNS-01, wildcards, automation requirement)
- [Certbot user guide](https://eff-certbot.readthedocs.io/en/stable/using.html) (DNS plugins, manual plugin, cert paths)
- [Let's Encrypt first IP address certificate announcement](https://letsencrypt.org/2025/07/01/issuing-our-first-ip-address-certificate.html) (IP certs exist since July 2025, gradual rollout)
- [Docker Compose services reference](https://docs.docker.com/reference/compose-file/services/) (ports long syntax, host_ip binding)
- [letsencrypt.org/blog/](https://letsencrypt.org/blog/) (IP certs GA in Certbot as of March 2026)

## TL;DR recipe

- Compose: one `eclipse-mosquitto:2.0.22-openssl` container. Mount config/data/log; publish
  8883 (TLS, public) and 1883 bound to host loopback only. Websockets listener not needed
  for v0: no browser client exists, both pets speak plain MQTT over TCP, and the
  websockets listener is built on libwebsockets ([mosquitto.conf(5): protocol](https://mosquitto.org/man/mosquitto-conf-5.html)).
- mosquitto.conf: two listeners (1883 local ops, 8883 TLS public), `allow_anonymous false`,
  `password_file` per-device accounts, `persistence true` + `persistence_location`,
  offline queue caps `max_queued_messages` / `max_queued_bytes`, session reaper
  `persistent_client_expiration 14d`.
- Client: `umqtt.simple` (frozen in official ESP32 MicroPython firmware) with an
  app-owned reconnect loop with exponential backoff. NOT `umqtt.robust2`: it does not
  exist in official micropython-lib; it is a third-party PyPI package (details in section 4).
- v0 certs: self-signed CA + server cert per mosquitto-tls(7), CA cert flashed to each pet,
  `cert_reqs=ssl.CERT_REQUIRED` + `cadata`. Upgrade path: buy a domain, certbot DNS-01.
  Let's Encrypt IP-address certs exist but are too new for v0 (section 5).

## 1. Compose essentials

Image: `eclipse-mosquitto:2.0.22-openssl`. The 2.0 series is built with openssl (TLS 1.3
and TLS-PSK support; the libressl variants omit those, per the docker [README](https://github.com/eclipse-mosquitto/mosquitto/blob/master/docker/README.md)).
The image runs as uid/gid 1883 (user `mosquitto`), expects config at
`/mosquitto/config/mosquitto.conf`, and declares `VOLUME ["/mosquitto/data", "/mosquitto/log"]`
([2.0-openssl Dockerfile](https://github.com/eclipse-mosquitto/mosquitto/blob/master/docker/2.0-openssl/Dockerfile)).

```yaml
# compose.yaml skeleton. No secrets inline: they live in .env (gitignored) or on the server.
services:
  broker:
    image: eclipse-mosquitto:2.0.22-openssl
    container_name: lovebird-broker
    restart: unless-stopped
    ports:
      # Public TLS listener for the two pets. The Compose spec: if host_ip is
      # not set, the port binds to all interfaces (0.0.0.0).
      - "8883:8883"
      # Plain MQTT for local ops/debugging from the host itself. Short syntax
      # "127.0.0.1:1883:1883" pins the HOST binding to loopback, so this port
      # is never reachable from the internet even though the listener inside
      # the container still listens on all container interfaces.
      - "127.0.0.1:1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log
    environment:
      - TZ=${TZ:-Europe/Berlin}
    mem_limit: 512m
```

Notes, each tied to a source:

- Port binding: the [Compose ports long syntax](https://docs.docker.com/reference/compose-file/services/)
  defines `host_ip`; if it is not set it binds all interfaces. The short form
  `127.0.0.1:1883:1883` is equivalent for the host-IP case. The 8883 publish binds all
  interfaces on the host; firewall it to just the two pet networks if the box runs
  anything else (out of scope here: server-side ops is a separate ticket).
- Volumes: the Dockerfile declares the data/log volumes and the README documents
  mounting config at `/mosquitto/config` (default CMD loads
  `/mosquitto/config/mosquitto.conf`). Bind-mounting all three keeps `mosquitto.db`
  (persistent sessions) and logs on the host, surviving container replacement.
- `restart: unless-stopped` and `mem_limit` are standard Compose service attributes.
  Mosquitto is lightweight; 512 MB is a guardrail on a 4 GB box, not a tuning knob.
  For a hard broker heap ceiling, mosquitto.conf offers `memory_limit`
  ([mosquitto.conf(5)](https://mosquitto.org/man/mosquitto-conf-5.html)).
- The image runs as uid/gid 1883. Host directories mounted at /mosquitto/{data,log}
  must be owned (or writable) by 1883 or the broker cannot write mosquitto.db. The
  container's [docker-entrypoint.sh](https://github.com/eclipse-mosquitto/mosquitto/blob/master/docker/2.0-openssl/docker-entrypoint.sh)
  chowns /mosquitto recursively when started as root, which covers the bind-mount case.
- Websockets: not needed in v0. The `protocol websockets` listener option exists in
  mosquitto.conf (MQTT tunneled over WebSockets) but serves browser clients, which v0
  does not have.
- Why pinned 2.0.22-openssl and not `2` or `latest`: those float to the 2.1 series,
  whose images ship a default conf with an extra http dashboard listener on 9883
  ([2.1-alpine/mosquitto.conf](https://github.com/eclipse-mosquitto/mosquitto/blob/master/docker/2.1-alpine/mosquitto.conf)).
  Pinning the exact tag keeps the port layout predictable and TLS 1.3 explicit.

## 2. mosquitto.conf directives (the answer)

Full annotated conf. Every directive is documented in [mosquitto.conf(5)](https://mosquitto.org/man/mosquitto-conf-5.html).

```conf
# lovebird broker conf. No secrets here, only file paths.

# --- listeners -----------------------------------------------------------
# Local ops listener: plain MQTT. Note the bind address here applies INSIDE
# the container (127.0.0.1 = container loopback). If you need to reach this
# from the host, rely on the compose port publish instead (127.0.0.1:1883:1883)
# and do not bind the listener to 127.0.0.1 inside the container, or publish
# it from the container's perspective (see note below the conf).
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd

# Public TLS listener for both pets.
listener 8883
protocol mqtt
certfile /mosquitto/certs/server.crt
keyfile /mosquitto/certs/server.key
cafile /mosquitto/certs/ca.crt
require_certificate false
allow_anonymous false
password_file /mosquitto/config/passwd
tls_version tlsv1.2

# --- globals --------------------------------------------------------------
# Mosquitto 2.x default: allow_anonymous false when listeners are defined
# (man page). Restated for self-documentation.
allow_anonymous false
password_file /mosquitto/config/passwd

# --- persistence and offline queueing (QoS 1 semantics) -------------------
persistence true
persistence_location /mosquitto/data/
# Flush session data every 10 minutes (default 1800s = 30 min), so a broker
# crash loses at most ~10 min of queued messages.
autosave_interval 600
# Per-client caps for QoS 1/2 messages queued while the client is offline.
# Defaults: 1000 messages, 0 bytes (no byte cap). "If both ... are specified,
# packets will be queued until the first limit is reached" (man page).
max_queued_messages 500
max_queued_bytes 1048576
# Reap persistent sessions (clean_session=0) that never reconnect.
# man page: duration is "an integer followed by one of s h d w m y".
persistent_client_expiration 14d
# QoS 0 queueing for offline persistent clients is off by default and is a
# non-standard option (man page: queue_qos0_messages). Leave it off: all pet
# events are QoS 1, so nothing important is skipped.
# queue_qos0_messages false    (default, implicit)

# --- logging ----------------------------------------------------------------
log_dest stdout
log_type error
log_type warning
log_type notice
log_type information
connection_messages true

# --- optional hardening ---------------------------------------------------
# memory_limit 268435456      # 256 MiB hard broker heap cap (man page)
# sys_interval 10             # $SYS updates; default 10s (man page)
```

Notes on the two-listener layout:

- `listener 1883` here has no bind address, so it listens on all container interfaces;
  the compose publish `127.0.0.1:1883:1883` is what restricts it to the host. This is
  the correct pairing for "local ops from the host". The man page's `listener port [bind]`
  form supports binding inside the container (e.g. `listener 1883 127.0.0.1` would make
  it unreachable from the published port, so do NOT use that form for the ops listener
  if you also publish it).
- `allow_anonymous false` + `password_file`: 2.x refuses remote anonymous connections
  by default (see the [docker README](https://github.com/eclipse-mosquitto/mosquitto/blob/master/docker/2.0-openssl/README.md):
  "Mosquitto 2.0 requires you to configure listeners and authentication before it
  will allow connections from anything other than the loopback interface").
  Per-device accounts: `mosquitto_passwd -c /mosquitto/config/passwd pet-one`, then
  `mosquitto_passwd /mosquitto/config/passwd pet-two`
  ([mosquitto_passwd(1)](https://mosquitto.org/man/mosquitto_passwd-1.html); hash
  default argon2id, `-b` batch mode exists but puts the password in shell history).
- `persistence true` + `persistence_location /mosquitto/data/`: the man page writes
  connection, subscription and message data to `mosquitto.db` at this path and reloads
  it on restart. Without it, a broker restart drops every queued message for offline pets.
- `max_queued_messages 500` / `max_queued_bytes 1048576`: per-client caps for QoS 1/2
  messages queued while the client is offline. When a cap is hit, further messages are
  silently dropped, so monitor queue depth via $SYS (sys_interval default 10s). For a
  pet publishing tiny JSON gestures at human pace, 500 messages is weeks of backlog.
- `persistent_client_expiration 14d`: the man page warns persistent sessions with
  random client ids linger forever; with stable per-pet client ids this directive is a
  simple reaper for decommissioned devices.
- `tls_version tlsv1.2`: sets the minimum TLS version for the listener. Default allows
  TLS 1.2 and 1.3; pinning the minimum documents intent and still permits 1.3.

## 3. QoS 1 queueing semantics for an offline pet (MQTT 3.1.1 spec)

- Persistent session: "If CleanSession is set to 0, the Server MUST resume
  communications with the Client based on state from the current Session... After the
  disconnection of a Session that had CleanSession set to 0, the Server MUST store
  further QoS 1 and QoS 2 messages that match any subscriptions" ([spec 3.1.2.4, MQTT-3.1.2-4/5](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  So the sleeping pet connects with clean_session=0 and a stable client id, the awake
  pet publishes QoS 1, and the broker queues until the sleeper reconnects.
- Session takeover: "If the ClientId represents a Client already connected to the
  Server then the Server MUST disconnect the existing Client" ([spec, MQTT-3.1.4-2](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  A pet rebooting while its old socket is half-open just takes over its own session.
  Two devices must never share a client id.
- Session Present: on reconnect, the CONNACK Session Present bit says whether the
  broker still had the session ([spec 3.2.2.2](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  umqtt.simple surfaces this: `connect()` returns True when a persistent session was
  resumed ([umqtt.simple README](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.simple/README.rst)).
- QoS 1 is "at least once": duplicates are possible, so the receiving pet must treat
  events idempotently (tag each gesture with a message id and ignore replays).
- Ordering: "A Server MUST by default treat each Topic as an Ordered Topic... It MUST
  send PUBLISH packets to consumers (for the same Topic and QoS) in the order that they
  were originally sent" ([spec 4.6, MQTT-4.6.0-5](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  Hugs arrive in the order they were sent.
- message_expiry_interval: **this is an MQTT v5 per-publish property, not an MQTT 3.1.1
  feature and not a mosquitto.conf directive.** Verified two ways: the mosquitto.conf(5)
  man page contains zero occurrences of `message_expiry_interval` (its closest
  neighbor, `retain_expiry_interval`, governs sweep timing for retained messages that
  carry a v5 expiry property), and umqtt.simple speaks protocol level 4 (MQTT 3.1.1),
  as its connect packet bytes show (`msg = bytearray(b"\x04MQTT\x04\x02\0\0")` in
  [simple.py](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.simple/umqtt/simple.py)).
  With v3.1.1 there is no per-message expiry: a queued message waits as long as the
  session lives. The v0 answers are queue caps (max_queued_messages, max_queued_bytes)
  plus a freshness timestamp in the payload if stale events must be ignored.
- Retained presence topics: "Retained messages do not form part of the Session state
  in the Server, they MUST NOT be deleted when the Session ends" ([spec, MQTT-3.1.2.7](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  Publish presence (online/offline) retained on `lovebird/<pet>/presence` so the other
  pet gets the last known state immediately on subscribe: "When a new subscription is
  established, the last retained message, if any, is delivered" ([spec 3.3.1.3, MQTT-3.3.1-5](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  To clear a retained topic, publish a zero-length retained message.
- Last Will is the presence mechanism: "If the Will Flag is set to 1... The Will
  Message MUST be published when the Network Connection is subsequently closed"
  ([spec 3.1.2.5](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)), which
  includes keepalive expiry. umqtt.simple: call `set_last_will(topic, msg, retain, qos)`
  before `connect()` ([simple.py](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.simple/umqtt/simple.py)).
- Keepalive: "If the Keep Alive value is non-zero and the Server does not receive a
  Control Packet from the Client within one and a half times the Keep Alive time
  period, it MUST disconnect" ([spec, MQTT-3.1.2-24](https://docs.oasis-open.org/mqtt/mqtt/v3.1.1/mqtt-v3.1.1.html)).
  With keepalive=60, the broker drops a silent pet after 90 s. Firmware must send
  something at least every 60 s; umqtt.simple has `ping()` for that and `wait_msg()`
  processes PINGRESP, but note it never sends PINGREQ itself: the application owns the
  keepalive cadence.

## 4. MicroPython client choice

**Verdict: `umqtt.simple` with an app-owned reconnect loop.**

Findings:

1. `umqtt.robust2` does not exist in official micropython-lib. The complete tree of
   [micropython-lib](https://github.com/micropython/micropython-lib) contains only
   `micropython/umqtt.simple` and `micropython/umqtt.robust` (verified via the GitHub
   git/trees API, recursive listing, not truncated). The name exists on
   [PyPI as micropython-umqtt.robust2](https://pypi.org/project/micropython-umqtt.robust2/),
   a third-party package by Wojciech Banaś (source: [fizista/micropython-umqtt.robust2](https://github.com/fizista/micropython-umqtt.robust2)),
   built on his own `umqtt.simple2` fork. Its README lists resumable sessions, offline
   message queueing, non-blocking operation and reconnect strategies, and warns it is
   larger than umqtt.robust (recommends .mpy compilation). Not part of the official
   distribution, single-maintainer: not recommended as a v0 dependency.
2. `umqtt.robust` (official) is a small subclass of umqtt.simple that adds automatic
   reconnect on OSError: [robust.py](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.robust/umqtt/robust.py)
   loops `self.connect(False)` (resuming the persistent session) with a fixed 2 s
   delay, and `publish()`/`wait_msg()`/`check_msg()` catch OSError, reconnect and
   retry. Its own [README](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.robust/README.rst)
   is candid that it is "a barebones example of how to reconnect", not a resilience
   framework, and that dumb retry loops drain batteries. The fixed 2 s delay and the
   blocking retries inside publish() do not fit a pet that should sleep for minutes
   between attempts.
3. `umqtt.simple` is frozen into the official ESP32 MicroPython firmware:
   [ports/esp32/boards/manifest.py](https://github.com/micropython/micropython/blob/master/ports/esp32/boards/manifest.py)
   includes `require("umqtt.simple")` and `require("umqtt.robust")`. Zero flash cost and
   no on-device dependency management. Feature set (from its README and source):
   QoS 0 and 1 for publish and subscribe (QoS 2 deliberately unsupported to keep size
   small), `clean_session` as the only connect option, `set_last_will`, TLS via
   `ssl=True` + `ssl_params`. Memory design: the README's stated requirement #1 is
   "Memory efficiency"; the module is ~7 KB of source and the subscribed-message
   callback design avoids an explicit message queue.
4. TLS on ESP32-S3: MicroPython's `ssl.wrap_socket(sock, cert_reqs, cadata,
   server_hostname)` exists and maps onto an SSLContext
   ([ssl docs](https://docs.micropython.org/en/latest/library/ssl.html)). Crucial
   mbedTLS note from those docs: "for mbedtls based ports, ssl.CERT_NONE and
   ssl.CERT_OPTIONAL will not validate any certificate, only ssl.CERT_REQUIRED will".
   The ESP32 port is mbedTLS-based, so a self-signed broker cert REQUIRES shipping the
   CA cert on the device and passing `cert_reqs=ssl.CERT_REQUIRED` with
   `cadata=<PEM>`. The bundled [example_pub_tls.py](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.simple/example_pub_tls.py)
   shows the pattern (it passes only server_hostname and notes certificate checking is
   disabled in that example; for lovebird, do it properly with cadata).

### Example connect code (v0 pattern)

```python
# pet_mqtt.py, MicroPython on ESP32-S3. umqtt.simple is frozen in firmware.
import ssl
import time
from umqtt.simple import MQTTClient

# Broker host, device username and password live in config_local.py on the
# device (gitignored in this repo). Placeholder values here.
BROKER = "broker.example.net"
PORT = 8883
CLIENT_ID = b"lovebird-pet-one"   # stable per device, never random (clean_session=0)
USER = "pet-one"
PASSWORD = "from-config-local"

with open("/lib/ca.crt") as f:    # CA that signed the broker cert, flashed per-device
    CA_PEM = f.read()

PRESENCE = b"lovebird/pet-one/presence"
TOPIC_IN = b"lovebird/pet-two/#"


def connect_pet():
    c = MQTTClient(
        CLIENT_ID,
        BROKER,
        PORT,
        user=USER,
        password=PASSWORD,
        keepalive=60,
        ssl=True,
        ssl_params={
            "server_hostname": BROKER,
            "cert_reqs": ssl.CERT_REQUIRED,
            "cadata": CA_PEM,
        },
    )
    # Last Will: retained offline flag so the other pet learns we died
    # (spec 3.1.2.5). Must be set before connect().
    c.set_last_will(PRESENCE, b'{"state":"offline"}', retain=True, qos=1)
    # Persistent session: broker queues QoS 1 messages while this pet sleeps
    # (spec 3.1.2.4, MQTT-3.1.2-5).
    resumed = c.connect(clean_session=False)
    return c, resumed


def main():
    c = None
    backoff = 5
    while True:
        try:
            if c is None:
                c, resumed = connect_pet()
                c.set_callback(lambda topic, msg: handle(topic, msg))
                c.publish(PRESENCE, b'{"state":"online"}', retain=True, qos=1)
                c.subscribe(TOPIC_IN, qos=1)
            # Blocking read: delivers subscribed messages to the callback and
            # processes PINGRESP. umqtt.simple never sends PINGREQ itself, so
            # the app must publish or ping at least every keepalive seconds.
            c.wait_msg()
            time.sleep_ms(50)
        except OSError:
            try:
                c.disconnect()
            except Exception:
                pass
            c = None
            time.sleep(backoff)
            backoff = min(backoff * 2, 300)   # 5, 10, 20, 40 ... capped at 300 s
        except Exception:
            # Non-network error: reset the client but do not hammer the broker.
            if c is not None:
                try:
                    c.disconnect()
                except Exception:
                    pass
            c = None
            time.sleep(30)
        else:
            backoff = 5


def handle(topic, msg):
    # QoS 1 is at-least-once: drop duplicates by message id in the payload.
    pass
```

Why an app-owned loop instead of umqtt.robust: the robust wrapper hides disconnects
behind blocking retries with a fixed 2 s delay inside publish(), which suits a
sensor posting readings more than a toy that should back off and let the user see
the offline state. Owning the loop lets firmware choose "retry now", "wait 5
minutes", or "deep sleep then retry", which is the robust README's own conclusion
(robustness is application-specific and umqtt.robust is a barebones example).

The QoS 1 publish path in umqtt.simple blocks until the matching PUBACK arrives
(it loops wait_msg() until it sees packet type 0x40 with the same packet id, per
[simple.py](https://github.com/micropython/micropython-lib/blob/master/micropython/umqtt.simple/umqtt/simple.py)),
so a failed publish surfaces as an OSError from the socket layer, which the app
loop catches and treats as a disconnect.

## 5. Certificates on a headless cloud box: domain vs self-signed for v0

| Option | What it takes | Pros | Cons | Verdict |
|---|---|---|---|---|
| A. Self-signed CA (mosquitto-tls recipe) | openssl anywhere: CA, server key+cert, ship ca.crt to each pet | No domain, no DNS API, no renewal automation needed, works day one | Manual renewal; no public PKI trust | **v0 pick** |
| B. Let's Encrypt + domain + DNS-01 | Buy a domain; certbot with a DNS plugin (dns-rfc2136, dns-cloudflare, ...) | Real CA, auto-renewal, hostname validation, free | Domain cost (~10 euro/yr); DNS API credentials on the box | **v1 upgrade** |
| C. Let's Encrypt IP-address cert | LE began issuing IP certs [July 2025](https://letsencrypt.org/2025/07/01/issuing-our-first-ip-address-certificate.html); [blog](https://letsencrypt.org/blog/) shows IP certs in Certbot as of March 2026 | No domain needed | Very new; gradual rollout; 6-day profiles; client support untested on ESP32 mbedTLS | skip for v0 |

Recommendation for v0: **option A**. Generate per [mosquitto-tls(7)](https://mosquitto.org/man/mosquitto-tls-7.html):
a CA (`openssl req -new -x509 -days <duration> -extensions v3_ca -keyout ca.key -out ca.crt`),
a server key and CSR signed by that CA, CN = the broker hostname (the man page: "When
prompted for the CN, please enter either your server (or broker) hostname or domain name").
The trust anchor is the CA cert flashed onto each pet; with `cert_reqs=CERT_REQUIRED` +
`cadata`, the device validates the chain exactly as it would a public CA's.

DNS-01 fits the headless box because it needs no inbound port 80 ([LE docs](https://letsencrypt.org/docs/challenge-types/)):
"Since automation of issuance and renewals is really important, it only makes sense to
use DNS-01 challenges if your DNS provider has an API you can use to automate updates".
Certbot ships DNS plugins (the [user guide](https://eff-certbot.readthedocs.io/en/stable/using.html)
lists dns-cloudflare, dns-google, dns-rfc2136 and more, each with a credentials file and
propagation-seconds option); certificates land in
`/etc/letsencrypt/live/<name>/fullchain.pem` and `privkey.pem`. The upgrade step: point
`certfile/keyfile` at those, drop the custom `cafile` (public CA chain validation), and
on each pet drop the custom `cadata` in favour of... note: MicroPython ESP32 does not
ship a system CA bundle by default; the practical v1 step is to keep shipping the ISRG
Root X1 PEM as `cadata`, or ship the fullchain. That detail should be verified on
hardware when the domain is bought.

Secrets hygiene: passwords and the CA key never enter the repo. `.env` is gitignored
(add the entry if missing); `mosquitto_passwd` writes `mosquitto/config/passwd` on the
server (that path is mounted into the container; keep it out of the repo or under
`secrets/`, which is already gitignored); per-device usernames/passwords go in
`config_local.py` on each pet (gitignored). The repo is public: no credentials, IPs or
personal data in anything pushed or posted, per repo policy.

## 6. Facts that could not be established from primary sources

- Whether the third-party `umqtt.robust2` is actively maintained: its GitHub README
  and PyPI metadata exist, but no first-party MicroPython source blesses it, and its
  last release date is not stated in the metadata we could fetch. Treated as
  unverified; not recommended for v0.
- MicroPython ESP32-S3 TLS handshake RAM cost: the MicroPython ssl docs do not
  publish per-port mbedTLS memory figures, so the handshake footprint on this exact
  Waveshare board could not be settled from primary sources. Measure on hardware
  (gc.mem_free() before/after connect) when boards arrive.
- Whether Let's Encrypt IP-address certificates validate on the ESP32 mbedTLS stack:
  not documented by MicroPython or LE. Avoided in the v0 recommendation for this reason.
- Exact renewal story for option A (self-signed): the mosquitto-tls(7) man page gives
  generation commands but no renewal workflow; this is a manual "regenerate and
  re-flash ca.crt + server cert" operation, which v0 accepts as a cost.

## 7. Checklist for the implementer

- [ ] Generate CA + server cert per mosquitto-tls(7) (CN = broker hostname or IP).
- [ ] Create per-device accounts: `mosquitto_passwd -c config/passwd pet-one` and
      `mosquitto_passwd config/passwd pet-two` (add a probe account only if you want
      the healthcheck to authenticate; otherwise skip it).
- [ ] Write mosquitto.conf as section 2; mount certs at /mosquitto/certs; chown the
      mounted data/log dirs to 1883:1883 (the entrypoint does it if the container
      starts as root).
- [ ] Flash ca.crt to each pet; put broker host + device username/password in
      config_local.py on each pet.
- [ ] Firmware: umqtt.simple, clean_session=False, stable client id,
      set_last_will(retain=True, qos=1), retained presence publishes, QoS 1 for
      gestures, idempotent duplicate handling, app-owned keepalive (ping() at < 60 s).
- [ ] Verify offline queueing: pet B offline, publish 3 hugs from pet A, reconnect
      pet B, confirm 3 deliveries in order (spec 4.6).
- [ ] Verify session takeover: reboot pet A mid-connection; the broker drops the old
      connection (MQTT-3.1.4-2) and A resumes its session.
- [ ] v1: buy a domain, certbot DNS-01, swap certs, update cadata on pets.