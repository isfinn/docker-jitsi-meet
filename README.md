# Jitsi Meet on Docker

![](resources/jitsi-docker.png)

[Jitsi] is a set of Open Source projects that allows you to easily build and deploy secure
videoconferencing solutions.

[Jitsi Meet] is a fully encrypted, 100% Open Source videoconferencing solution that you can use
all day, every day, for free — with no account needed.

This repository contains the necessary tools to run a Jitsi Meet stack on [Docker] using
[Docker Compose].

**NOTE: This setup is experimental.**

## Table of contents

* [Quick start](#quick-start)
* [Architecture](#architecture)
  - [Images](#images)
  - [Design considerations](#design-considerations)
* [Configurations](#configuration)
  - [Advanced configuration](#advanced-configuration)
  - [Running on a LAN environment](#running-on-a-lan-environment)
* [Limitations](#limitations)

<hr />

## Quick start

In order to quickly run Jitsi Meet on a machine running Docker and Docker Compose,
follow these steps:

* Create a ``.env`` file by copying and adjusting ``env.example``.
* Run ``docker-compose up -d``.
* Access the web UI at ``https://localhost:8443`` (or ``http://localhost:8000`` for HTTP, or
  a different port, in case you edited the compose file).

If you want to use jigasi too, first configure your env file with SIP credentials
and then run Docker Compose as follows: ``docker-compose -f docker-compose.yml -f jigasi.yml up -d``

If you want to enable TURN server, configure it and run Docker Compose as
follows: ``docker-compose -f docker-compose.yml -f turn.yml up``

## Architecture

A Jitsi Meet installation can be broken down into the following components:

* A web interface
* An XMPP server
* A conference focus component
* A video router (could be more than one)
* A SIP gateway for audio calls

![](resources/docker-jitsi-meet.png)

The diagram shows a typical deployment in a host running Docker. This project
separates each of the components above into interlinked containers. To this end,
several container images are provided.

### Images

* **base**: Debian stable base image with the [S6 Overlay] for process control and the
  [Jitsi repositories] enabled. All other images are based off this one.
* **base-java**: Same as the above, plus Java (OpenJDK).
* **web**: Jitsi Meet web UI, served with nginx.
* **prosody**: [Prosody], the XMPP server.
* **jicofo**: [Jicofo], the XMPP focus component.
* **jvb**: [Jitsi Videobridge], the video router.
* **jigasi**: [Jigasi], the SIP (audio only) gateway.
* **turn**: [Coturn], the TURN server.

### Design considerations

Jitsi Meet uses XMPP for signalling, thus the need for the XMPP server. The setup provided
by these containers does not expose the XMPP server to the outside world. Instead, it's kept
completely sealed, and routing of XMPP traffic only happens on a user defined network.

The XMPP server can be exposed to the outside world, but that's out of the scope of this
project.

## Configuration

The configuration is performed via environment variables contained in a ``.env`` file. You
can copy the provided ``env.example`` file as a reference.

**IMPORTANT**: At the moment, configuration is not regenerated on every container boot, so
if you make any changes to your ``.env`` file, make sure you remove the configuration directory
before starting your containers again.

Variable | Description | Example
--- | --- | ---
`CONFIG` | Directory where all configuration will be stored | /opt/jitsi-meet-cfg
`TZ` | System Time Zone | Europe/Amsterdam
`HTTP_PORT` | Exposed port for HTTP traffic | 8000
`HTTPS_PORT` | Exposed port for HTTPS traffic | 8443
`DOCKER_HOST_ADDRESS` | IP address of the Docker host, needed for LAN environments | 192.168.1.1
`PUBLIC_URL` | Public url for the web service | https://meet.example.com

**NOTE**: The mobile apps won't work with self-signed certificates (the default)
see below for instructions on how to obtain a proper certificate with Let's Encrypt.

### Let's Encrypt configuration

If you plan on exposing this container setup to the outside traffic directly and
want a proper TLS certificate, you are in luck because Let's Encrypt support is
built right in. Here are the required options:

Variable | Description | Example
--- | --- | ---
`ENABLE_LETSENCRYPT` | Enable Let's Encrypt certificate generation | 1
`LETSENCRYPT_DOMAIN` | Domain for which to generate the certificate | meet.example.com
`LETSENCRYPT_EMAIL` | E-Mail for receiving important account notifications (mandatory) | alice@atlanta.net

In addition, you will need to set `HTTP_PORT` to 80 and `HTTPS_PORT` to 443.

### SIP gateway configuration

If you want to enable the SIP gateway, these options are required:

Variable | Description | Example
--- | --- | ---
`JIGASI_SIP_URI` | SIP URI for incoming / outgoing calls | test@sip2sip.info
`JIGASI_SIP_PASSWORD` | Password for the specified SIP account | passw0rd
`JIGASI_SIP_SERVER` | SIP server (use the SIP account domain if in doubt) | sip2sip.info
`JIGASI_SIP_PORT` | SIP server port | 5060
`JIGASI_SIP_TRANSPORT` | SIP transport | UDP

### JItsi BRoadcasting Infrastructure configuration

For working jibri, you need setup alsa loopback on the host:

#### for Centos 7, module already compiled with kernel, so just run:
```
# configure 5 capture/playback interfaces
echo "options snd-aloop enable=1,1,1,1,1 index=0,1,2,3,4" > /etc/modprobe.d/asound.conf
# setup autoload the module
echo "snd_aloop" > /etc/modules-load.d/snd_aloop.conf
# load the module
modprobe snd-aloop
# check that the module is loaded
lsmod | grep snd_aloop
```
#### for Ubuntu 16.04 (Xenial):
```
# install the module
apt update && apt install linux-image-extra-virtual
# configure 5 capture/playback interfaces
echo "options snd-aloop enable=1,1,1,1,1 index=0,1,2,3,4" > /etc/modprobe.d/asound.conf
# setup autoload the module
echo "snd-aloop" >> /etc/modules
# check that the module is loaded
lsmod | grep snd_aloop
```
If you want to enable the JIBRI, these options are required:

Variable | Description | Example
--- | --- | ---
`ENABLE_RECORDING` | Enable recording conference to local disk | 1

### Transcription configuration

If you want to enable the Transcribing function, these options are required:

Variable | Description | Example
--- | --- | ---
`ENABLE_JIGASI_TRANSCRIBER` | Enable Jigasi transcription in a conference | 1
`GOOGLE_APPLICATION_CREDENTIALS` | Credentials for connect to Cloud Google API from Jigasi. Path located inside the container | /config/key.json

For set `GOOGLE_APPLICATION_CREDENTIALS` please read https://cloud.google.com/text-to-speech/docs/quickstart-protocol section "Before you begin" from 1 to 5 paragraph

### Authentication

Authentication can be controlled with the environment variables below. If guest
access is enabled, unauthenticated users will need to wait until a user authenticates
before they can join a room. If guest access is not enabled, every user will need
to authenticate before they can join.

Variable | Description | Example
--- | --- | ---
`ENABLE_AUTH` | Enable authentication | 1
`ENABLE_GUESTS` | Enable guest access | 1
`ENABLE_LDAP_AUTH` | Enable authentication via LDAP. Depended from `ENABLE_AUTH` | 1

Variables that might be configured if the `ENABLE_LDAP_AUTH` is set:

Variable | Description | Example
--- | --- | ---
`LDAP_URL` | URL for ldap connection | ldaps://ldap.domain.com/
`LDAP_BASE` | LDAP base DN. Can be empty. | DC=example,DC=domain,DC=com
`LDAP_BINDDN` | LDAP user DN. Do not specify this parameter for the anonymous bind. | CN=binduser,OU=users,DC=example,DC=domain,DC=com
`LDAP_BINDPW` | LDAP user password. Do not specify this parameter for the anonymous bind. | LdapUserPassw0rd
`LDAP_FILTER` | LDAP filter. | (sAMAccountName=%u)
`LDAP_AUTH_METHOD` | LDAP authentication method. | bind
`LDAP_VERSION` | LDAP protocol version | 3
`LDAP_USE_TLS` | Enable LDAP TLS | 1
`LDAP_TLS_CIPHERS` | Set TLS ciphers list to allow | SECURE256:SECURE128
`LDAP_TLS_CHECK_PEER` | Require and verify LDAP server certificate | 1

Internal users must be created with the ``prosodyctl`` utility in the ``prosody`` container.
In order to do that, first execute a shell in the corresponding container:

``docker-compose exec prosody /bin/bash``

Once in the container, run the following command to create a user:

``prosodyctl --config /config/prosody.cfg.lua register user meet.jitsi password``

### Advanced configuration

These configuration options are already set and generally don't need to be changed.

Variable | Description | Default value
--- | --- | ---
`XMPP_DOMAIN` | Internal XMPP domain | meet.jitsi
`XMPP_AUTH_DOMAIN` | Internal XMPP domain for authenticated services | auth.meet.jitsi
`XMPP_SERVER` | Internal XMPP server name xmpp.meet.jitsi | xmpp.meet.jitsi
`XMPP_BOSH_URL_BASE` | Internal XMPP server URL for BOSH module | http://xmpp.meet.jitsi:5280
`XMPP_MUC_DOMAIN` | XMPP domain for the MUC | muc.meet.jitsi
`XMPP_INTERNAL_MUC_DOMAIN` | XMPP domain for the internal MUC | internal-muc.meet.jitsi
`XMPP_GUEST_DOMAIN` | XMPP domain for unauthenticated users | guest.meet.jitsi
`XMPP_RECORDER_DOMAIN` | Domain for the jibri recorder | recorder.meet.jitsi
`XMPP_MODULES` | Custom Prosody modules for XMPP_DOMAIN (comma separated) | mod_info,mod_alert
`XMPP_MUC_MODULES` | Custom Prosody modules for MUC component (comma separated) | mod_info,mod_alert
`XMPP_INTERNAL_MUC_MODULES` | Custom Prosody modules for internal MUC component (comma separated) | mod_info,mod_alert
`JICOFO_COMPONENT_SECRET` | XMPP component password for Jicofo | s3cr37
`JICOFO_AUTH_USER` | XMPP user for Jicofo client connections | focus
`JICOFO_AUTH_PASSWORD` | XMPP password for Jicofo client connections | passw0rd
`JVB_AUTH_USER` | XMPP user for JVB MUC client connections | jvb
`JVB_AUTH_PASSWORD` | XMPP password for JVB MUC client connections | passw0rd
`JVB_STUN_SERVERS` | STUN servers used to discover the server's public IP | stun.l.google.com:19302, stun1.l.google.com:19302, stun2.l.google.com:19302
`JVB_PORT` | UDP port for media used by Jitsi Videobridge | 10000
`JVB_TCP_HARVESTER_DISABLED` | Disable the additional harvester which allows video over TCP (rather than just UDP) | true
`JVB_TCP_PORT` | TCP port for media used by Jitsi Videobridge when the TCP Harvester is enabled | 4443
`JVB_BREWERY_MUC` | MUC name for the JVB pool | jvbbrewery
`JVB_ENABLE_APIS` | Comma separated list of JVB APIs to enable | none
`JIGASI_XMPP_USER` | XMPP user for Jigasi MUC client connections | jigasi
`JIGASI_XMPP_PASSWORD` | XMPP password for Jigasi MUC client connections | passw0rd
`JIGASI_BREWERY_MUC` | MUC name for the Jigasi pool | jigasibrewery
`JIGASI_PORT_MIN` | Minimum port for media used by Jigasi | 20000
`JIGASI_PORT_MAX` | Maximum port for media used by Jigasi | 20050
`JIGASI_ENABLE_SDES_SRTP` | Enable SDES srtp | 1
`JIGASI_SIP_KEEP_ALIVE_METHOD` | Keepalive method | OPTIONS
`JIGASI_HEALTH_CHECK_SIP_URI` | Health-check extension. Jigasi will call it for healthcheck | keepalive
`JIGASI_HEALTH_CHECK_INTERVAL` | Interval of healthcheck in milliseconds | 300000
`JIBRI_RECORDER_USER` | Internal recorder user for Jibri client connections | recorder
`JIBRI_RECORDER_PASSWORD` | Internal recorder password for Jibri client connections | passw0rd
`JIBRI_RECORDING_DIR` | Directory for recordings inside Jibri container | /config/recordings
`JIBRI_FINALIZE_RECORDING_SCRIPT_PATH` | The finalizing script. Will run after recording is complete | /config/finalize.sh
`JIBRI_XMPP_USER` | Internal user for Jibri client connections. | jibri
`JIBRI_RECORDER_PASSWORD` | Internal user for Jibri client connections | passw0rd
`JIBRI_STRIP_DOMAIN_JID` | Prefix domain for strip inside Jibri (please see env.example for details) | muc
`JIBRI_BREWERY_MUC` | MUC name for the Jibri pool | jibribrewery
`JIBRI_PENDING_TIMEOUT` | MUC connection timeout | 90
`JIBRI_LOGS_DIR` | Directory for logs inside Jibri container | /config/logs
`JIBRI_EXTERNAL_INSTANCE` | Set only if the jibri hosted on a different host | 1
`JIGASI_TRANSCRIBER_RECORD_AUDIO` | Jigasi will recordord an audio when transcriber is on | true
`JIGASI_TRANSCRIBER_SEND_TXT` | Jigasi will send transcribed text to the chat when transcriber is on | true
`JIGASI_TRANSCRIBER_ADVERTISE_URL` | Jigasi post to the chat an url with transcription file | true
`DISABLE_HTTPS` | Disable HTTPS, this can be useful if TLS connections are going to be handled outside of this setup | 1
`ENABLE_HTTP_REDIRECT` | Redirects HTTP traffic to HTTPS | 1
`ENABLE_CHROME_SCREEN_SHARING` | Enable screensharing for Chrome (has been working in Chrome since version 72) | 1
`START_WITH_VIDEO_MUTED` | Mute a video when user is coming to a conference | 1
`CALENDAR_MS_APP_ID` | Enable Microsoft calendar integarion. Set Azure application ID | 00000000-0000-0000-0000-000040240063
`ETHERPAD_URL_BASE` | Set etherpad-lite URL | http://etherpad:9001
`ENABLE_SPEAKER_STATS` | Enable speaker statistics. Before enable it, make sure that modules from project jitsi-meet/resources/prosody-plugins successfully loads to /prosody-plugins-custom | 1

#### Setting up Octo (cascaded bridges)
NOTE: For get working octo properly you have to set header "X-User-Region" before it passing to nginx. It can be realized via geoip or another logic and it's not described here.

The header "X-User-Region" will be passed through nginx and dynamically set variable for `userRegion` in config.js file via ssi nginx module.
If `userRegion` and `JVB_OCTO_REGION` the same region, user will be connected to the instanse JVB that has this region.

This behavion already preconfigured and receiving X-User-Region in config.js looks like this:

```
deploymentInfo: {
    // shard: "shard1",
    // region: europe" -->',
    userRegion: '<!--#echo var="http_x_user_region" default="us-east-1" -->'
},
```

If you want to enable the Octo cascading briges, these options are required:

Variable | Description | Default value
--- | --- | ---
`JICOFO_BRIDGE_SELECTION_STRATEGY` | Bridge selection stratagy for new connections | RegionBasedBridgeSelectionStrategy
`JVB_OCTO_BIND_PORT` | The UDP port number which the Octo relay should use | 4096
`JVB_OCTO_REGION` | The region that the jitsi-videbridge instance is in | us-east-1

The brige selection stratagy is:

* `SplitBridgeSelectionStrategy` - can be used for testing. It tries to select a new bridge for each client, regardless of the regions.

* `RegionBasedBridgeSelectionStrategy` - matches the region of the clients to the region of the Jitsi Videobridge instances. Used by default.

### TURN(S) server
For enable turn server for P2P and JVB connections, please add to the variable `GLOBAL_MODULES` string `turncredentials` and set variables below

Variable | Description | Default value
--- | --- | ---
`TURN_ENABLE` | Use TURN for P2P and JVB (bridge mode) connections | 0
`TURN_REALM` | Realm to be used for the users with long-term credentials mechanism or with TURN REST API | realm
`TURN_SECRET` | Secret for connect to TURN server | keepthissecret
`TURN_TYPE` | Type of TURN(s) (turn/turns) | turns
`TURN_HOST` | Annonce FQDN/IP address of the turn server via XMPP (XEP-0215) | 192.168.1.1
`TURN_PUBLIC_IP` | Public IP address for an instance of turn server | set dynamically
`TURN_PORT` | TLS/TCP/UDP turn port for connection | 5349
`TURN_TRANSPORT` | transport for turn connection (tcp/udp) | tcp
`TURN_RTP_MIN` | RTP start port for turn/turns connections | 10000
`TURN_RTP_MAX` | RTP end port for turn/turns connections | 11000


For enable web-admin panel for turn, please set variables below

Variable | Description | Default value
--- | --- | ---
`TURN_ADMIN_ENABLE` | Enable web-admin panel | 0
`TURN_ADMIN_USER` | Username for admin panel | admin
`TURN_ADMIN_SECRET` | Password for admin panel | changeme
`TURN_ADMIN_PORT` | HTTP(s) port for acess to admin panel | 8443

### Running on a LAN environment

If running in a LAN environment (as well as on the public Internet, via NAT) is a requirement,
the ``DOCKER_HOST_ADDRESS`` should be set. This way, the Videobridge will advertise the IP address
of the host running Docker instead of the internal IP address that Docker assigned it, thus making [ICE]
succeed.

The public IP address is discovered via [STUN]. STUN servers can be specified with the ``JVB_STUN_SERVERS``
option.

## TODO

* Support container replicas (where applicable).
* Docker Swarm mode.
* More services:
  * Jibri.

[Jitsi]: https://jitsi.org/
[Jitsi Meet]: https://jitsi.org/jitsi-meet/
[Docker]: https://www.docker.com
[Docker Compose]: https://docs.docker.com/compose/
[Swarm mode]: https://docs.docker.com/engine/swarm/
[S6 Overlay]: https://github.com/just-containers/s6-overlay
[Jitsi repositories]: https://jitsi.org/downloads/
[Prosody]: https://prosody.im/
[Jicofo]: https://github.com/jitsi/jicofo
[Jitsi Videobridge]: https://github.com/jitsi/jitsi-videobridge
[Jigasi]: https://github.com/jitsi/jigasi
[ICE]: https://en.wikipedia.org/wiki/Interactive_Connectivity_Establishment
[STUN]: https://en.wikipedia.org/wiki/STUN
[Coturn]: https://github.com/coturn/coturn
