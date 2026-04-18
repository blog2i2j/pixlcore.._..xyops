# Tailscale

## Overview

xyOps pairs well with [Tailscale](https://tailscale.com/) (specifically [Tailscale Serve](https://tailscale.com/kb/1312/serve)), which acts as an SSO auth system by forwarding trusted headers.  This document describes how to set up xyOps on your Tailnet using the Tailscale Serve CLI, and also as a "sidecar" with Docker Compose.

First, please read the [xyOps SSO Guide](sso.md), as it introduces several concepts we will be referencing here.

## Header Map

Here is how you should configure your [SSO Header Map](sso.md#header-map) for Tailscale Serve use:

```json
"header_map": {
	"username": "Tailscale-User-Login",
	"full_name": "Tailscale-User-Name",
	"email": "Tailscale-User-Login",
	"avatar": "Tailscale-User-Profile-Pic",
	"groups": "Tailscale-App-Capabilities"
}
```

Alternatively, you can simply enable SSO and set the `preset` property to `tailscale`, which configures the header map (among other things) for you.  Example with environment variables:

```sh
XYOPS_SSO__enabled="true"
XYOPS_SSO__preset="tailscale"
```

## Usernames

Please read the [SSO Header Cleanup](sso.md#header-cleanup) section regarding username cleanup, so you understand how your Tailscale username (which comes in as an email address) will be translated on the xyOps side.  By default, the email domain is stripped, and your username becomes the first part of your email address.

## Admin Bootstrap

If you want to just get up and running quickly, you can use the [SSO Admin Bootstrap](sso.md#admin-bootstrap) feature to automatically promote yourself to a full administrator.  Just note that this is designed as a one-time setup shortcut, and will log a loud warning in the xyOps activity log on every login.

## Capabilities

Tailscale can forward along what they call "capabilities", which can translate to roles and privileges on the xyOps side.  In short, you can use Tailscale's capability grants to automatically assign your xyOps users appropriate privileges and/or roles.  Here is how to set it up:

1. Login to your [Tailscale Admin Console](https://login.tailscale.com/welcome).
2. Click on the "Access Controls" tab.
3. Click on "JSON Editor".

Locate the `grants` array in the JSON editor, or create it if needed.  Here is an example:

```json
"grants": [
	{
		"src": ["autogroup:admin"],
		"dst": ["autogroup:admin"],
		"app": {
			"xyops.io/cap/ts": [ {"privileges": ["admin"], "roles": []} ],
		}
	}
]
```

This will automatically promote anyone in your `autogroup:admin` Tailscale group to a full xyOps administrator.  The `autogroup:admin` group should already exist in your account, as it is automatically created for you by Tailscale.  You can see all your groups, and also also add new groups, by clicking on the "Visual Editor" tab, and then the "Groups" sub-tab.

The `app` section of the grant JSON is very important -- the syntax must be exact:

```json
"app": {
	"xyops.io/cap/ts": [ {"privileges": ["admin"], "roles": []} ],
}
```

The `xyops.io/cap/ts` is a unique key that xyOps looks for when it parses the incoming headers.  It must be set exactly to `xyops.io/cap/ts`, and the value must be an array of objects, as shown above.  The properties inside the object are passed directly to xyOps:

- The `privileges` sub-array can contain any valid [xyOps Privilege IDs](privileges.md), e.g. `admin`.
- The `roles` sub-array can contain any valid [xyOps Role IDs](users.md#roles), which are user-created.

Note that by default, roles and privileges are applied "additively" to user records.  Meaning, the SSO sync will never *remove* a role or privilege.  This is so you can manually apply your own user roles and permissions using the xyOps Admin UI, and everything plays nice.  However, if you do not want this behavior, and instead want Tailscale to be the single source of truth for all user roles and privileges, set the SSO [replace_roles](sso.md#configuration) and/or [replace_privileges](sso.md#configuration) properties to `true`.  Those will replace **all** the user roles and/or privileges with whatever we get from the Tailscale capabilities trusted header.  This sync happens on every user login and session refresh, wiping out any local changes made in xyOps.

Make sure you click the "Save" button when you are finished editing the JSON in your Tailscale Admin Console.

## Base URL

xyOps needs to know the base URL of the hosted application so it can generate self-referencing URLs in things like outbound emails and web hooks.  To do this, you'll need to set the [base_app_url](config.md#base_app_url) configuration property to your Tailscale-provided app URL.  The hostname will be the current machine name, followed by your custom Tailnet domain.  Example:

```json
"base_app_url": "https://joemax.taild89302.ts.net"
```

Or via environment variable:

```sh
XYOPS_base_app_url="https://joemax.taild89302.ts.net"
```

Additionally, if you plan on adding remote satellite servers, xyOps needs to know what hostname to use for advertising itself to its cluster (i.e. how satellite servers will connect back to the primary conductor server).  To do this, add a `hostname` top-level configuration property set to your machine's Tailnet hostname:

```json
"hostname": "joemax.taild89302.ts.net"
```

Or via environment variable:

```sh
XYOPS_hostname="joemax.taild89302.ts.net"
```

By default xyOps uses the current machine's local hostname for this, so we want to override it to use our special Tailnet hostname.

## HTTPS

Assuming you have [Tailscale HTTPS](https://login.tailscale.com/admin/dns) enabled on your Tailnet (highly recommended), this means that Tailscale Serve will **only** support HTTPS URLs to your app.  So, we need to set a couple of extra configuration properties so you can add satellite servers and have them communicate over HTTPS as well:

- `satellite.config.secure`: `true`
- `satellite.config.port`: `443`

Or via environment variables:

```sh
XYOPS_satellite__config__secure="true"
XYOPS_satellite__config__port="443"
```

## Serve

Once you have xyOps configured and running with SSO enabled, start [Tailscale Serve](https://tailscale.com/kb/1312/serve) using these command-line arguments:

```sh
tailscale serve --accept-app-caps=xyops.io/cap/ts 5522
```

The special `--accept-app-caps=xyops.io/cap/ts` argument tells Tailscale to forward the special `Tailscale-App-Capabilities` HTTP header with all incoming requests, which xyOps uses to apply user privileges and roles (see [Capabilities](#capabilities) above).

Note that the first time you visit your Tailscale-provided secure HTTPS URL for your served app, there may be a short delay as Tailscale provisions the TLS certificate.  If you receive a timeout error, please wait a few seconds and refresh.  This is normal.

## Logout

When a user clicks the "Logout" button in the xyOps UI, the user's session data and cookie are deleted.  However, with an SSO provider like Tailscale we cannot "fully" log the user out, because they will still be connected and authenticated with their Tailnet, and thus if they just navigate their browser back to the app, they'll instantly log back in (this is by design).

So, for Tailscale SSO the best way to handle this is to set the [logout_url](sso.md#logging-out) to the following value, which displays a message to the user describing the "partial logout" situation:

```sh
XYOPS_SSO__logout_url="/api/app/sso_logout"
```

Note that if you use the `preset` feature to enable Tailscale (see above), the `logout_url` is set automatically for you.

## Security

For security hardening, set your [SSO IP Whitelist](sso.md#ip-whitelist) to only accept trusted headers from localhost, as that is how Tailscale Serve routes traffic:

```json
"whitelist": ["127.0.0.1", "::1/128"]
```

The whitelist can also be specified via environment variable.  Just use a CSV list of IPs and/or CIDRs in that case:

```sh
XYOPS_SSO__whitelist="127.0.0.1,::1/128"
```

Finally, remember to **delete the stock admin user account** that xyOps automatically creates on first install.  Since you are using SSO this account is technically "unreachable", but deleting it is safest as it has an insecure password by default.

## Sidecar

Tailscale has a really cool feature which allows xyOps to run in a Docker container as its own dedicated "node" on your Tailnet.  This is done by running Tailscale as a [sidecar container](https://tailscale.com/blog/docker-tailscale-guide) alongside a xyOps container.  Tailscale then handles everything including DNS, TLS, and proxying requests to xyOps, including forwarding trusted headers for automatic login and privilege / role assignments.  This guide contains everything you need to get up and running.

### Tailscale Admin Setup

The first thing you'll need to do is login to your [Tailscale Admin Console](https://login.tailscale.com/welcome), and then:

1. Read the [Capabilities](#capabilities) section above, to add a grant into your Tailnet policy file, so xyOps can automatically assign privileges and roles to your users.
2. Create an Auth Key.  See [Generate An Auth Key]( https://tailscale.com/kb/1085/auth-keys#generate-an-auth-key) for instructions.

Remember, auth keys expire by default.  See [Tailscale Key Expiry](https://tailscale.com/docs/features/access-control/auth-keys#key-expiry) to learn more.  For permanent access, you can disable key expiry, and add a tag to the machine.

### Host Setup

Make sure you have [Docker](https://www.docker.com/) and the Docker CLI with Docker Compose all installed and running.

Create a directory on your host machine for xyOps to use, and `cd` into it.  It just needs to store a couple of configuration files and some volume-mapped directories for the containers.

### Env File

Create an `.env` file in our host directory with the following contents:

```sh
# Tailscale Configuration
TS_AUTHKEY="YOUR_TAILSCALE_AUTHKEY"
TS_HOST="xyops.taild89302.ts.net"
TZ="America/Los_Angeles"
```

- Change the `TS_AUTHKEY` value to your own auth key you just created on your Tailscale console.
- Change the `TS_HOST` value to your own Tailnet domain name, leaving in the `xyops.` prefix.  See the [DNS](https://login.tailscale.com/admin/dns) tab in your Tailscale Admin Console.
- Change the `TZ` to your own local timezone, so xyOps can rotate logs and reset daily stats at "your" midnight.

### Docker Compose

Create a `compose.yaml` file in our host directory with the following contents:

```yaml
configs:
  ts-serve:
    content: |
      {"TCP":{"443":{"HTTPS":true}},
      "Web":{"$${TS_CERT_DOMAIN}:443":
          {"Handlers":{"/":
          {"Proxy":"http://127.0.0.1:5522","AcceptAppCaps":["xyops.io/cap/ts"]}}}},
      "AllowFunnel":{"$${TS_CERT_DOMAIN}:443":false}}

services:
  # Tailscale Sidecar Configuration
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale-xyops # Name for local container management
    hostname: xyops # Name used within your Tailscale environment
    environment:
      TS_AUTHKEY: "${TS_AUTHKEY}"
      TS_STATE_DIR: "/var/lib/tailscale"
      TS_SERVE_CONFIG: "/config/serve.json"
      TS_USERSPACE: "false"
      TS_ENABLE_HEALTH_CHECK: "true" # Enable healthcheck endpoint: "/healthz"
      TS_LOCAL_ADDR_PORT: "127.0.0.1:41234" # The <addr>:<port> for the healthz endpoint
      TS_ACCEPT_DNS: "true" # Use Tailscale MagicDNS
      TS_AUTH_ONCE: "true"
    configs:
      - source: ts-serve
        target: /config/serve.json
    volumes:
      - ./ts-config:/config # Config folder used to store Tailscale files
      - ./ts-state:/var/lib/tailscale # Tailscale requirement
    devices:
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - net_admin
    healthcheck:
      test: ["CMD", "wget", "--spider", "-q", "http://127.0.0.1:41234/healthz"]
      interval: 1m
      timeout: 10s
      retries: 3
      start_period: 10s
    restart: unless-stopped

  xyops01:
    image: ghcr.io/pixlcore/xyops:latest
    container_name: xyops01
    network_mode: service:tailscale # Use Sidecar
    init: true
    restart: unless-stopped
    environment:
      XYOPS_xysat_local: "true"
      XYOPS_hostname: "${TS_HOST}"
      XYOPS_masters: "${TS_HOST}"
      XYOPS_base_app_url: "https://${TS_HOST}"
      XYOPS_satellite__config__secure: "true"
      XYOPS_satellite__config__port: "443"
      XYOPS_SSO__enabled: "true"
      XYOPS_SSO__preset: "tailscale"
      XYOPS_SSO__whitelist: "127.0.0.1,::1/128"
      TZ: "${TZ}"
    volumes:
      - xy-data:/opt/xyops/data
      - ./xyops01-conf:/opt/xyops/conf
      - /var/run/docker.sock:/var/run/docker.sock
    depends_on:
      tailscale:
        condition: service_healthy

volumes:
  xy-data:
```

You should not need to edit anything in the `compose.yaml` file, but here are a couple of notes:

- The `XYOPS_xysat_local` environment variable causes xyOps to launch [xySat](hosting.md#satellite) in the background, in the same container.  This is so you can start running jobs right away -- it is great for testing and home labs, but *not recommended for production*.
- The `/var/run/docker.sock` bind is optional, and allows xyOps to launch its own containers (i.e. for the [Docker Plugin](plugins.md#docker-plugin), and the [Plugin Marketplace](marketplace.md)).

### Start Up

Type this command to start everything up (the `-d` switch runs it in the background):

```sh
docker compose up -d
```

And then visit your `TS_HOST` URL in your favorite browser.  Example:

https://xyops.taild89302.ts.net/

Note that the first time you visit the URL, there may be a short delay as Tailscale provisions the TLS certificate.
