# MCP for TestRail Server: Getting Started

This article takes you from nothing to an agent talking to your TestRail Server through the MCP server. It takes about ten minutes. Replace `https://testrail.corp.example` with the URL of your TestRail Server instance as the container will reach it, including the path if TestRail is mounted under one (for example `https://testrail.corp.example/testrail`).

## 1. Prepare TestRail

1. Make sure the TestRail API is enabled: Administration, Site Settings, API, "Enable API".
2. Every user who will connect an agent generates an API key on their My Settings page, API Keys tab. The MCP server never stores keys: each request carries the user's email and key, and TestRail applies that user's permissions.

## 2. Load the image

The image comes as a download, one file per CPU architecture, each with a checksum file next to it:

| File                                            | For                                     |
| ----------------------------------------------- | --------------------------------------- |
| `testrail-mcp-x.y.z-linux-amd64.tar.gz`         | Most Linux servers and Intel machines   |
| `testrail-mcp-x.y.z-linux-arm64.tar.gz`         | ARM servers and Macs with Apple silicon |
| `testrail-mcp-x.y.z-linux-<arch>.tar.gz.sha256` | The checksum of the file beside it      |

`uname -m` on the Docker host tells you which one you need: `x86_64` is `amd64`; `aarch64` or `arm64` is `arm64`. Download that file and its `.sha256` into the same folder, check that the download is intact, and load it into Docker:

```zsh
sha256sum -c testrail-mcp-x.y.z-linux-amd64.tar.gz.sha256   # macOS: shasum -a 256 -c ...
docker load -i testrail-mcp-x.y.z-linux-amd64.tar.gz
```

`docker load` prints the two tags the image carries, `testrail/mcp:x.y.z` and `testrail/mcp:x.y`. Nothing is downloaded from a registry: the compose bundle runs the image you loaded.

## 3. Start the server with Docker Compose

1. Download and unpack the compose bundle, or create a folder with the `docker-compose.yml` and `.env.example` files from it.
2. Copy `.env.example` to `.env` and set `TESTRAIL_ON_PREM_INSTANCE` to your instance URL. The bundle already names the image release line (`x.y`) it was published with, which is one of the tags you loaded, so nothing else is required.
3. Start it:

```zsh
docker compose up -d
docker compose logs -f
```

The log shows one line when the server is ready:

```
TestRail MCP starting in on-prem mode: instance https://testrail.corp.example, API key auth only
```

If Compose answers `pull access denied for testrail/mcp` instead, it did not find the loaded image under the tag it expects, so it tried to download it: run `docker image ls testrail/mcp` and check that the `x.y` tag in `docker-compose.yml` is listed, or load the image again (step 2).

Without Compose, the equivalent `docker run` is:

```zsh
docker run -d --name testrail-mcp --restart unless-stopped \
  -p 8000:8000 \
  -e TESTRAIL_ON_PREM_INSTANCE=https://testrail.corp.example \
  testrail/mcp:x.y.z
```

## 4. Check that it runs

```zsh
curl http://localhost:8000/health
```

answers `200`. Then open `http://<server-host>:8000/` in a browser: the documentation page shows your instance URL in its form, already filled in and read-only.

If the container exits instead, `docker compose logs` tells you why. The two startup errors are a missing or malformed `TESTRAIL_ON_PREM_INSTANCE`; see Upgrading and troubleshooting.

## 5. Connect an agent

The documentation page at the server's root URL is the place to do this, because it generates the exact command for your instance and server address:

1. Enter your TestRail email and API key in the form. They stay in the browser; the page only uses them to fill in the snippets below it.
2. Pick your MCP host (Claude Code, Claude Desktop, Codex CLI, Cursor, GitHub Copilot in VS Code, Zed, Windsurf, or a generic MCP host).
3. Run the command or paste the configuration it shows.

For Claude Code, for example, the page produces a command of this shape:

```zsh
claude mcp add testrail "http://<server-host>:8000/mcp?INSTANCE-URL=https://testrail.corp.example" \
  --transport http --scope user \
  --header "Authorization: Basic <base64 of email:api-key>"
```

Run `claude mcp list` to confirm it shows as connected, then ask the agent something simple such as "list my TestRail projects".

## 6. Put it behind TLS

Before anyone outside a test machine uses it, publish the server through your reverse proxy with TLS, and have the proxy forward the `X-Forwarded-Proto` and `X-Forwarded-Host` headers so the documentation page generates `https://` URLs. The Configuration article has a sample proxy block.

## More documentation

- [MCP for TestRail Server: Overview](docs/01-overview.md)
- [MCP for TestRail Server: Configuration](docs/03-configuration.md)
- [MCP for TestRail Server: Upgrading and Troubleshooting](docs/04-upgrading-and-troubleshooting.md)
