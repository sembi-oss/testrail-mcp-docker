# MCP for TestRail Server: Upgrading and Troubleshooting

## Upgrading

The server keeps no state, so an upgrade is a new container from a newer image. Agents are asked to run `check_mcp_version` at the start of each session, which tells you whether a newer release exists and how to install it. Each image download carries two tags:

| Tag     | Points at                                       | Use it for                                                            |
| ------- | ----------------------------------------------- | --------------------------------------------------------------------- |
| `x.y.z` | that release, always                            | Reproducible deployments: set `TESTRAIL_MCP_VERSION=x.y.z` in `.env`. |
| `x.y`   | the newest release of that line you have loaded | The compose bundle's default: loading a patch release moves it.       |

These tags version the MCP server, not TestRail. Upgrading the image does not touch your TestRail instance, and upgrading TestRail does not require a new image: the server detects the new version and offers the tools it supports. A newer image is what adds tools for TestRail features released after the image you run (see "Hidden tools" below).

With the compose bundle, each release's `docker-compose.yml` names its own `x.y` line. For a patch release, download its image file for your architecture, check and load it as in Getting started, and recreate the container; Compose sees that `x.y` now points at the new image:

```zsh
sha256sum -c testrail-mcp-x.y.z-linux-amd64.tar.gz.sha256   # macOS: shasum -a 256 -c ...
docker load -i testrail-mcp-x.y.z-linux-amd64.tar.gz
docker compose up -d
```

For a new release line, load its image the same way, then download that release's bundle and replace `docker-compose.yml` with its copy (your `.env` is untouched), or set `TESTRAIL_MCP_VERSION` in `.env` to the line or exact version you want, and run `docker compose up -d`. `.env` wins over the bundle's default, so clear it again when you next replace the compose file.

Older images stay in Docker after an upgrade, so going back is setting `TESTRAIL_MCP_VERSION` to the earlier `x.y.z` and running `docker compose up -d`. Remove one you no longer need with `docker image rm testrail/mcp:x.y.z`.

Agents reconnect on their next request; nothing has to be reconfigured on their side. The release notes on each GitHub release say when a release needs a newer TestRail Server version than the one before, because tools that need a newer TestRail are hidden rather than failing (see "Hidden tools" below).

## Which version is running

Any of these answers it:

- `docker compose ps` shows the image tag; `docker inspect --format '{{index .Config.Labels "org.opencontainers.image.version"}}' <container>` prints the version baked into the image.
- The server card at `http://<server-host>:8000/.well-known/mcp/server-card.json` reports it under `serverInfo.version`, with the access tier after a `+` (for example `1.2.0+readwrite`).
- The documentation page carries it in a `<meta name="mcp-server-version">` tag, and agents receive it in the MCP `initialize` handshake.
- Ask the agent: `get_mcp_version` returns it, and `check_mcp_version` also says whether a newer release exists.

## Hidden tools

The server detects the TestRail Server version when a request needs it (through the `get_version` API on TestRail 10.7.1 and newer, from the login page before that) and hides the tools that TestRail version does not support, instead of offering tools that would fail. The documentation page marks these tools with the version they need (for example `10.9+`). The server logs the decision:

```
tool gating: TestRail version unknown for testrail.corp.example; 94/107 tools advertised (new tools hidden)
```

"Version unknown" means the instance could not be reached or did not answer as expected. The server asks again by itself, so there is no need to restart the container: a failed detection is retried after about 30 seconds, and a successful one is reused for 10 minutes. After you upgrade TestRail, tools the new version supports appear on the agent's next tool listing within about ten minutes. If the version stays unknown, check the instance URL and that the container can reach it. Ask the agent to run `check_testrail_version`: it reports the detected version, where it came from, and whether a newer TestRail Server release exists.

## Common errors

| Symptom                                                                                                                                                  | Cause                                                                                                                   | Fix                                                                                          |
| -------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- |
| Container exits at once; log says `TESTRAIL_ON_PREM_INSTANCE must be set: this image only runs against an on-prem TestRail instance`                     | The variable is missing.                                                                                                | Set it in `.env` (Compose) or with `-e` (`docker run`).                                      |
| Container exits at once; log says `TESTRAIL_ON_PREM_INSTANCE must be http(s)://host[:port][/path] with a plain /path prefix and no query or credentials` | The value has a query string, user name, or an odd path.                                                                | Use the form `https://host[:port][/path]`.                                                   |
| Agent gets `401 Missing Authorization`                                                                                                                   | The MCP URL was registered without the `Authorization: Basic` header.                                                   | Re-run the command from the documentation page with the email and API key filled in.         |
| Agent gets `400 INSTANCE-URL does not match the configured instance`                                                                                     | The URL carries an `INSTANCE-URL` for a different host than the server is configured for.                               | Drop the parameter or set it to the configured instance; the page generates the right value. |
| Tool calls fail with a `401` or `403` from TestRail                                                                                                      | Wrong API key, the API is disabled, or the user lacks the permission in TestRail.                                       | Check Administration, Site Settings, API, and the user's key and role.                       |
| Tool calls fail with a connection error naming the instance                                                                                              | The container cannot reach TestRail, or the URL is wrong (a common one is a missing `/testrail` path).                  | Probe the instance from inside the container, see below.                                     |
| Tool calls fail with `CERTIFICATE_VERIFY_FAILED`                                                                                                         | TestRail's certificate is signed by an internal or self-signed certificate authority that the container does not trust. | Give the container your CA, see "Trusting an internal certificate authority".                |
| Documentation page shows `http://` URLs or the wrong host behind the proxy                                                                               | The proxy does not forward `X-Forwarded-Proto` and `X-Forwarded-Host`.                                                  | Add the headers (see Configuration).                                                         |
| Agent reports fewer tools than the documentation page lists                                                                                              | Version gating, see "Hidden tools".                                                                                     | Upgrade TestRail, or accept that the newer tools are absent.                                 |
| A long tool call stops with a timeout error                                                                                                              | The call exceeded `SEMBI_TOOL_TIMEOUT_SECONDS`, or the proxy's read timeout is shorter.                                 | Raise the variable, and the proxy timeout with it.                                           |

## Trusting an internal certificate authority

The container trusts a standard list of public certificate authorities, and nothing else. If TestRail is served over https with a certificate from your own CA, every call fails certificate verification until the container is given that CA. Put the CA certificate in a PEM file, mount it into the container, and point `SSL_CERT_FILE` at it.

The setting replaces the built-in list rather than adding to it, so the file must also hold any public CA the container still needs. The only other places the container connects are TestRail's update-check endpoint and the compose bundle's `VERSION` file on GitHub. If you leave the public CAs out, `check_testrail_version` falls back to the version shipped with the image and `check_mcp_version` reports the latest release as unknown; to skip both requests, set `TESTRAIL_VERSION_CHECK_URL=off` and `TESTRAIL_MCP_VERSION_CHECK_URL=off`.

With Compose, add a `docker-compose.override.yml` next to `docker-compose.yml` (Compose reads it automatically; the fixed variable list in the main file does not carry `SSL_CERT_FILE`):

```yaml
services:
  testrail-mcp:
    environment:
      SSL_CERT_FILE: /certs/ca-bundle.pem
    volumes:
      - ./ca-bundle.pem:/certs/ca-bundle.pem:ro
```

With `docker run`, add `-e SSL_CERT_FILE=/certs/ca-bundle.pem -v "$PWD/ca-bundle.pem:/certs/ca-bundle.pem:ro"`. The container runs as a non-root user, so the file must be readable by everyone.

## Probing the instance from the container

The container has no shell tools beyond Python, so use it to ask TestRail's API for the project list without credentials. A reachable instance with the API enabled answers `401`; a `404` means the path is wrong (often a missing `/testrail`), a `403` usually means the API is disabled, and a connection error means the container cannot reach the host at all.

```zsh
docker compose exec testrail-mcp python -c '
import os, urllib.error, urllib.request
url = os.environ["TESTRAIL_ON_PREM_INSTANCE"].rstrip("/") + "/index.php?/api/v2/get_projects"
try:
    print(urllib.request.urlopen(url, timeout=10).status)
except urllib.error.HTTPError as error:
    print(error.code)
'
```

## Checking what you loaded

The images are not signed yet. They are provided only by Sembi, and each image file comes with a `.sha256` file holding its checksum: `sha256sum -c` (on macOS `shasum -a 256 -c`) with that file answers `OK` only for an intact download. Get both files from the same place Sembi sent you to, and do not load a file whose check fails.

Once loaded, the image ID identifies exactly what a tag points at, and the version label says which release it is:

```zsh
docker image inspect --format '{{.Id}}' testrail/mcp:x.y.z
docker image inspect --format '{{index .Config.Labels "org.opencontainers.image.version"}}' testrail/mcp:x.y.z
```

Comparing the ID before and after you load a file shows whether `x.y` moved, and `docker compose ps` together with `docker inspect --format '{{.Image}}' <container>` shows which image a running container uses.
