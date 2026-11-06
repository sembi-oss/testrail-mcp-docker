# TestRail MCP Server on-prem: getting started

This article takes you from nothing to an agent talking to your TestRail Server through the MCP server. It takes about ten minutes. Replace `https://testrail.corp.example` with the URL of your TestRail instance as the container will reach it, including the path if TestRail is mounted under one (for example `https://testrail.corp.example/testrail`).

## 1. Prepare TestRail

1. Make sure the TestRail API is enabled: Administration, Site Settings, API, "Enable API".
2. Every user who will connect an agent generates an API key on their My Settings page, API Keys tab. The MCP server never stores keys: each request carries the user's email and key, and TestRail applies that user's permissions.

## 2. Start the server with Docker Compose

1. Download and unpack the compose bundle, or create a folder with the `docker-compose.yml` and `.env.example` files from it.
2. Copy `.env.example` to `.env` and set `TESTRAIL_ON_PREM_INSTANCE` to your instance URL. The bundle already names the image release line it was published with, so nothing else is required.
3. Start it:

```zsh
docker compose up -d
docker compose logs -f
```

The log shows one line when the server is ready:

```
TestRail MCP starting in on-prem mode: instance https://testrail.corp.example, API key auth only
```

Without Compose, the equivalent `docker run` is:

```zsh
docker run -d --name testrail-mcp --restart unless-stopped \
  -p 8000:8000 \
  -e TESTRAIL_ON_PREM_INSTANCE=https://testrail.corp.example \
  testrail/mcp:latest
```

## 3. Check that it runs

```zsh
curl http://localhost:8000/health
```

answers `200`. Then open `http://<server-host>:8000/` in a browser: the documentation page shows your instance URL in its form, already filled in and read-only.

If the container exits instead, `docker compose logs` tells you why. The two startup errors are a missing or malformed `TESTRAIL_ON_PREM_INSTANCE`; see Upgrading and troubleshooting.

## 4. Connect an agent

The documentation page at the server's root URL is the place to do this, because it generates the exact command for your instance and server address:

1. Enter your TestRail email and API key in the form. They stay in the browser; the page only uses them to fill in the snippets below it.
2. Pick your agent host (Claude Code, Claude Desktop, Codex CLI, Cursor, GitHub Copilot in VS Code, Zed, Windsurf, or a generic MCP host).
3. Run the command or paste the configuration it shows.

For Claude Code, for example, the page produces a command of this shape:

```zsh
claude mcp add testrail "http://<server-host>:8000/mcp?INSTANCE-URL=https://testrail.corp.example" \
  --transport http --scope user \
  --header "Authorization: Basic <base64 of email:api-key>"
```

Run `claude mcp list` to confirm it shows as connected, then ask the agent something simple such as "list my TestRail projects".

## 5. Put it behind TLS

Before anyone outside a test machine uses it, publish the server through your reverse proxy with TLS, and have the proxy forward the `X-Forwarded-Proto` and `X-Forwarded-Host` headers so the documentation page generates `https://` URLs. The Configuration article has a sample proxy block.

## More documentation

- [TestRail MCP Server on-prem: overview](docs/01-overview.md)
- [TestRail MCP Server on-prem: configuration](docs/03-configuration.md)
- [TestRail MCP Server on-prem: upgrading and troubleshooting](docs/04-upgrading-and-troubleshooting.md)
