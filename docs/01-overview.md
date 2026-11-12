# MCP for TestRail Server: Overview

The on-prem TestRail MCP Server lets any MCP-host (Claude Code, Claude Desktop, Codex CLI, Cursor, GitHub Copilot in VS Code, Zed, Windsurf and others) work directly with TestRail Server. For TestRail Server customers it ships as a Docker container image that you run inside your own network, next to your TestRail Server instance. This article says what the on-prem server is, what it needs, and where to go next.

## What it is

- One stateless container. No database, no volumes, no configuration file: the only required setting is the URL of your TestRail Server instance.
- The same tool set as the hosted TestRail MCP Server for everything in the TestRail API: projects, suites, sections, cases, runs, plans, results, milestones, attachments, BDD, labels, shared steps, datasets and more. 
- There are three extra MCP tools: `check_testrail_version` reports whether your TestRail Server is on the latest release. `get_mcp_version` provides the version of the MCP container you're running. `check_mcp_version` reports whether your MCP container is on the latest release.
- A built-in documentation page at the server's root URL that explains, for each type of MCP host, how to connect to the MCP server. The page knows your instance URL and generates the exact command or configuration snippet for it.
- Every MCP request authenticates with the user's own TestRail email and API key, so TestRail's user-level permissions apply to everything the agent does.

## What it is not

- It does not include the Sembi IQ features of the hosted server (e.g., semantic case search, the coverage-gap analysis, test case generation, the agent plugin and skills). These depend on the Sembi IQ cloud service and are not available on-prem.
- It does not support OAuth sign-in. On-prem authentication is API key only.
- It does not terminate TLS. Put it behind the reverse proxy you already use for TestRail Server (see Configuration).

## Requirements

- A TestRail Server instance the container can reach over the network, with the API enabled (Administration, Site Settings, API). Every user who connects an agent needs an API key from their "My Settings" page.
- Docker Engine with the Compose plugin, or any container runtime that can run an OCI image. The image is provided for `linux/amd64` and `linux/arm64`.
- Outbound network access from the container to your TestRail Server instance. Nothing else is required: the optional check for the latest TestRail Server and MCP Server releases can be switched off for air-gapped sites.
- A reverse proxy for TLS between the agents and the server. The server speaks plain HTTP and every request carries a user's API key.

## Downloads

- Container image: `testrail/mcp`, as a download from Sembi: one `.tar.gz` file per architecture, each with a `.sha256` checksum file, that you load with `docker load`. Each one carries the tags `x.y.z` and `x.y`.
- Compose bundle (`docker-compose.yml`, `.env.example`, this documentation): <https://github.com/sembi-oss/testrail-mcp-docker>.

The image's version (`x.y.z`) is the MCP server's own, not a TestRail Server version, so you do not pick an image to match your TestRail Server. The server detects the version of your TestRail Server instance and provides the matching MCP tools that version does not support (see "Hidden tools" in the upgrading article).

## Continued

1. [Getting started](../README.md): load the image, run it, open the documentation page, connect an agent.
2. [Configuration](03-configuration.md): environment variables, ports, TLS, logging, access tiers.
3. [Upgrading and troubleshooting](04-upgrading-and-troubleshooting.md): pinning versions, finding out what is running, common errors.
