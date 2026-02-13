# Bonjour-Aware Reverse Proxy

A local reverse proxy that listens for Bonjour/mDNS service announcements and automatically
makes them accessible via `<service-name>.local` in the browser.

## Problem

When running local dev servers (e.g. webpack-dev-server), they start on arbitrary ports. You
end up with bookmarks like `https://localhost:8080`. Webpack-dev-server already supports
publishing Bonjour service records (e.g. `ocean-explorer._https._tcp.local`) for discovery,
but browsers can't use those — they need hostname resolution and a standard port.

## Idea

A lightweight proxy that:

1. **Discovers services** — Listens for Bonjour/mDNS service announcements on the local
   network (e.g. using the `bonjour-service` npm package).
2. **Publishes hostnames** — For each discovered service, publishes an mDNS A record so
   `<service-name>.local` resolves to the machine's IP.
3. **Reverse proxies** — Runs on port 80/443 and forwards requests based on the `Host` header
   to the correct `host:port` from the service announcement.

This would let you type `https://ocean-explorer.local` in a browser and reach the dev server
automatically — no `/etc/hosts` editing, no remembering port numbers.

## Existing webpack-dev-server support

webpack-dev-server already has a built-in
[`bonjour` option](https://webpack.js.org/configuration/dev-server/#devserverbonjour) that
publishes the dev server as a Bonjour service on startup. Configuration is minimal:

```js
devServer: {
  bonjour: {
    name: 'ocean-explorer',
  },
}
```

This publishes a service record like `ocean-explorer._https._tcp.local` with the server's
host and port. The service name, type, and subtypes can be customized via the options object
(passed through to the `bonjour-service` npm package). This is already enough for programmatic
discovery — for example, Playwright tests can find the dev server's port without hardcoding
it. The missing piece is turning that service announcement into something a browser can use
(hostname resolution + reverse proxy on a standard port), which is what this tool would
provide.

Vite does not have built-in bonjour support. If this proxy tool gained traction, it could
motivate adding similar service publishing to Vite — the implementation would be small since
`bonjour-service` does the heavy lifting. A Vite plugin could also work.

## Prior art

- **dinghy-http-proxy** — Did something similar for Docker containers (watched Docker events,
  configured nginx). The [dinghy](https://github.com/codekitchen/dinghy) project was archived
  in October 2024.
- **dory** — [Linux equivalent](https://github.com/FreedomBen/dory) of dinghy, also
  Docker-event-driven.
- **docker-mdns** — [Publishes `.local` hostnames](https://github.com/phyber/docker-mdns)
  for Docker containers via Avahi, but Linux-only and no proxy.
- **mdns-publisher** — Python tool for publishing mDNS CNAME aliases. Used for home lab
  setups alongside nginx. [Blog post](https://andrewdupont.net/2022/01/27/using-mdns-aliases-within-your-home-network/).

## Only one proxy per machine

A machine can only have one process bound to port 80/443 at a time. This means a
bonjour-only reverse proxy would conflict with other local proxies a developer might already
be running — for example, a Docker reverse proxy like dinghy-http-proxy or Traefik for
container-based services.

Rather than forcing developers to choose, this tool should also handle Docker container
proxying (watching Docker events, reading labels) so it can be the single reverse proxy on
the machine. Think of it as a successor to dinghy-http-proxy that adds Bonjour discovery
alongside Docker support.

The tool should also be really good about detecting port conflicts. If something is already
bound to port 80 or 443 when the proxy starts, it should clearly report what process holds
the port and how to resolve the conflict, rather than failing with a cryptic EADDRINUSE error.

## Other challenges

- **macOS port 5353** — The system `mDNSResponder` owns the mDNS port. Publishing additional
  hostnames from userspace needs care (CNAME records pointing to the machine's existing
  `.local` hostname may work).
- **Port 80/443 permissions** — Binding to low ports requires root or `CAP_NET_BIND_SERVICE`.
- **HTTPS certificates** — dinghy-http-proxy handled this with a self-signed CA; that
  approach can be reused. For npm-based projects we already use
  [`mkcert`](https://github.com/FiloSottile/mkcert) to generate locally-trusted certs, so
  integrating with or delegating to `mkcert` would fit existing developer workflows.
- **Windows** — Windows doesn't resolve multi-label `.local` names (e.g. `foo.bar.local`),
  but single-label names like `ocean-explorer.local` should work.

## Possible tech stack

- `bonjour-service` or `multicast-dns` for mDNS discovery and publishing
- `http-proxy` or `node-http-proxy` for reverse proxying
- Node.js CLI tool, run as a background service
