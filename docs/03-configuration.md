# MCP for TestRail Server: Configuration

Everything the on-prem server reads comes from environment variables. With the compose bundle you set them in `.env`; with `docker run` you pass them with `-e`.

## Environment variables

| Variable                         | Required | Default                             | Purpose                                                                                                                                                                                                                                          |
| -------------------------------- | -------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `TESTRAIL_ON_PREM_INSTANCE`      | yes      | none                                | The TestRail Server instance, as the container reaches it: `http(s)://host[:port][/path]`, no query string or credentials. Its presence is what puts the server in on-prem mode; the container refuses to start without it.                      |
| `MCP_LOG_FORMAT`                 | no       | `human`                             | `json` writes one JSON object per log line for a log shipper.                                                                                                                                                                                    |
| `SEMBI_TOOL_TIMEOUT_SECONDS`     | no       | `120`                               | Ceiling for one tool call. A call that takes longer is aborted with an error returned to the agent.                                                                                                                                              |
| `TESTRAIL_VERSION_CHECK_URL`     | no       | TestRail's update-check endpoint    | Where `check_testrail_version` asks for the latest TestRail Server release. `off` disables the request for air-gapped sites; the tool then answers with the release known when the image was built.                                              |
| `TESTRAIL_MCP_VERSION_CHECK_URL` | no       | Docker Hub's tag list for the image | Where `check_mcp_version` asks for the latest release of this MCP server. A replacement URL must answer in the format of Docker Hub's tag list. `off` disables the request for air-gapped sites; the tool then reports the running version only. |
| `MCP_HTTP_PORT`                  | no       | `8000`                              | Port the server listens on inside the container. With Compose, change the host port in `.env` (`TESTRAIL_MCP_PORT`) instead and leave this alone.                                                                                                |

The compose bundle adds three variables of its own, read only by `docker compose`: `TESTRAIL_MCP_VERSION` (an image tag other than the release line the bundle shipped with), `TESTRAIL_MCP_PORT` (the host port) and `TESTRAIL_MCP_IMAGE` (a mirror or registry proxy instead of Docker Hub).

## Endpoints

| Path                                | Purpose                                                                                          |
| ----------------------------------- | ------------------------------------------------------------------------------------------------ |
| `/`                                 | The documentation page: how to connect each agent host, and the list of tools.                   |
| `/mcp`                              | The MCP endpoint agents connect to (Streamable HTTP).                                            |
| `/health`                           | Answers `200` when the server is up; the image's own health check and your load balancer use it. |
| `/.well-known/mcp/server-card.json` | Machine-readable description of the server, including its version.                               |

## Authentication

Every request to `/mcp` carries an `Authorization: Basic` header holding `email:api-key` in base64, where the email is the user's TestRail login and the key is one of their TestRail API keys. The server forwards them to TestRail on each call, so what the agent can do is exactly what that user can do in TestRail. Nothing is stored.

The `INSTANCE-URL` query parameter that the hosted TestRail MCP Server requires is optional on-prem, and the documentation page still includes it so the snippets look the same. When it is present it must match the configured instance; a different value is rejected with `400 INSTANCE-URL does not match the configured instance`. The server can never be pointed at another host by a client.

## Access tiers

The `mode` query parameter on the MCP URL picks how much an agent may do:

| `mode`      | Tools                                                                                                                                                                                                                                                                                                                                                                         |
| ----------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `read-only` | Reads only: nothing in TestRail changes.                                                                                                                                                                                                                                                                                                                                      |
| (absent)    | Reads and writes: creating and updating projects, cases, runs, results and so on, and closing or archiving runs and plans. The default.                                                                                                                                                                                                                                       |
| `full`      | Everything in the default, plus the destructive tools: every `delete_*` tool (projects, suites, cases, sections, runs, plans, datasets, variables and attachments) and six update tools flagged as destructive, several of which replace a whole list at once (`update_milestone`, `update_plan`, `update_shared_step`, `update_test`, `update_tests` and `update_variable`). |

For example `http://<server-host>:8000/mcp?mode=read-only&INSTANCE-URL=https://testrail.corp.example`. The documentation page's tool list marks each tool as read-only, write or irreversible, which describes what the tool does and is not the `mode` that serves it: closing and archiving are marked irreversible, yet the default mode includes them. Keep `full` for agents you trust to delete.

## TLS and reverse proxies

The container speaks plain HTTP. Terminate TLS on the reverse proxy you already run for TestRail and forward to the container's port. Three headers matter. `X-Forwarded-Proto` and `X-Forwarded-Host` tell the server the public scheme and host, which the documentation page uses to build the URLs in its snippets. Without them, a page reached through `https://mcp.corp.example` would print `http://` commands with the container's internal host name. `X-Forwarded-For` only affects the caller address in the logs (see Logging). MCP responses are streamed, so the proxy must not buffer them. An nginx location block:

```
location / {
    proxy_pass         http://127.0.0.1:8000;
    proxy_http_version 1.1;
    proxy_set_header   Host              $host;
    proxy_set_header   X-Forwarded-Proto $scheme;
    proxy_set_header   X-Forwarded-Host  $host;
    proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_buffering    off;
    proxy_read_timeout 300s;
}
```

The read timeout should be at least `SEMBI_TOOL_TIMEOUT_SECONDS` so the proxy does not cut a long tool call before the server does.

The server logs the last address in `X-Forwarded-For` as the caller, which is the one your proxy appended (`$proxy_add_x_forwarded_for` does that). Entries further left are whatever the client sent. If the proxy does not set the header, the log shows the proxy's own address, and if the container's port is also reachable without the proxy, a client can put any address it likes in the header. Publish the port only to the proxy, for example `127.0.0.1:8000:8000` in the compose `ports` entry. The address is used for the log line only; it plays no part in who is allowed in.

## Network access

The container makes outbound connections to three places: your TestRail instance, on every tool call; TestRail's update-check endpoint (`https://secure.testrail.com/customers/testrail/version/get`), once a day at most, when `check_testrail_version` runs with `TESTRAIL_VERSION_CHECK_URL` left at its default; and Docker Hub's tag list for the image (`https://hub.docker.com/v2/namespaces/testrail/repositories/mcp/tags`), once a day at most, when `check_mcp_version` runs with `TESTRAIL_MCP_VERSION_CHECK_URL` left at its default. Set either variable to `off` to remove its connection. The documentation page loads fonts, icons and syntax highlighting from public CDNs in the user's browser; without internet access the page still works, with plain-text snippets and missing icons.

## Logging

The server logs to standard output, one line per tool call with the caller's address, user agent and a hashed identifier of the login email (never the email or the key), plus one line per rejected request. `docker compose logs -f` follows it.

With `MCP_LOG_FORMAT=json` the server's own lines are JSON objects with the same fields, for a log shipper. The web server underneath (uvicorn) still prints its own plain-text lines: startup and shutdown messages, and one access line per HTTP request, including the image's health check every 30 seconds. A shipper in JSON mode should accept lines that do not start with `{`, or drop them.

The image logs at info level: the per-call and rejection lines, plus warnings. For troubleshooting, `LOG_LEVEL=debug` adds diagnostic lines such as the TestRail version detection and its cache hits. The compose file passes a fixed list of variables to the container, so add it in a `docker-compose.override.yml` next to it (Compose reads that file automatically):

```yaml
services:
  testrail-mcp:
    environment:
      LOG_LEVEL: debug
```

With `docker run`, add `-e LOG_LEVEL=debug`.
