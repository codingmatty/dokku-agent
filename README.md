# dokku-agent

A [Dokku](https://dokku.com/) plugin to run [Docker Agent](https://docker.github.io/docker-agent/) on your server. It provides passthrough commands (`agent:run`, `agent:exec`) and **managed Agent API services** (similar to [dokku-postgres](https://github.com/dokku/dokku-postgres)): create named instances, link them to apps, and get `AGENT_API_URL` in the app environment.

## Install the plugin

```bash
# From Git (clone or add as submodule)
sudo dokku plugin:install https://github.com/codingmatty/dokku-agent.git agent

# Or from a local path
sudo dokku plugin:install /path/to/dokku-agent agent
```

If you see **Permission denied** when creating a service (e.g. `mkdir: cannot create directory '/var/lib/dokku/services/agent-api'`), the install trigger needs to run to create and chown the data directories. Either reinstall the plugin (run the `sudo dokku plugin:install` command again) or fix permissions once:

```bash
sudo mkdir -p /var/lib/dokku/services/agent-api /var/lib/dokku/data/agent
sudo chown -R dokku:dokku /var/lib/dokku/services/agent-api /var/lib/dokku/data/agent
```

(Use your Dokku system user/group if different from `dokku`.)

## Prerequisites

- **Docker** – Already present on Dokku hosts.
- **Docker Agent CLI** – Installed automatically when you install the plugin (the install trigger downloads the binary for the dokku user). Run `dokku agent:install` to verify; set API keys with `dokku agent:set-key`.

- **Model API key** – Set at least one provider key via the plugin (see below); no need to set it on the host environment.

## Managed Agent API services

Create named Agent API instances, link them to apps, and manage them like Postgres or Redis.

| Command                                                             | Description                                              |
| ------------------------------------------------------------------- | -------------------------------------------------------- |
| `dokku agent:api-create <service> [config-path]`                    | Create an Agent API service (optional agent.yaml path)   |
| `dokku agent:api-info <service> [--url\|--port\|--status\|--links]` | Show service info                                        |
| `dokku agent:api-link <service> <app>`                              | Link service to app (sets `AGENT_API_URL`)               |
| `dokku agent:api-unlink <service> <app>`                            | Unlink service from app                                  |
| `dokku agent:api-list`                                              | List all Agent API services                              |
| `dokku agent:api-start <service>`                                   | Start the API server                                     |
| `dokku agent:api-stop <service>`                                    | Stop the API server                                      |
| `dokku agent:api-destroy <service>`                                 | Destroy service (unlink first)                           |
| `dokku agent:api-expose <service> [port]`                           | Expose service on a port (optionally set port and start) |
| `dokku agent:api-unexpose <service>`                                | Stop service (no longer exposed)                         |
| `dokku agent:api-set-config <service> <path>`                       | Update agent.yaml from file and restart                  |

### Example workflow

```bash
# Create a service (default agent config) or with your own agent.yaml
dokku agent:api-create test-agent
dokku agent:api-create my-agent ./path/to/agent.yaml

# Link to an app; the app gets AGENT_API_URL (e.g. http://172.17.0.1:12345)
dokku agent:api-link test-agent agent-app

# Inspect, expose on a specific port, or update config
dokku agent:api-info test-agent
dokku agent:api-info test-agent --url
dokku agent:api-expose test-agent 8080
dokku agent:api-set-config test-agent ./new-agent.yaml

# Unlink and destroy
dokku agent:api-unlink test-agent agent-app
dokku agent:api-destroy test-agent
```

Each service gets its own port, config directory, session DB, and log files. Data lives under `$DOKKU_LIB_ROOT/services/agent-api/<service>/`.

## API keys (stored in plugin)

Keys are stored under `$DOKKU_LIB_ROOT/data/agent/keys.env` and are **automatically loaded** before every `docker agent` invocation (run, exec, and managed API services). You do not need to set them in the host environment.

| Command                             | Description                              |
| ----------------------------------- | ---------------------------------------- |
| `dokku agent:set-key <KEY> <value>` | Set a key (e.g. `ANTHROPIC_API_KEY`)     |
| `dokku agent:unset-key <KEY>`       | Remove a key                             |
| `dokku agent:keys`                  | List stored key names (values not shown) |

```bash
dokku agent:set-key ANTHROPIC_API_KEY sk-ant-...
dokku agent:set-key OPENAI_API_KEY sk-...
dokku agent:keys
```

## Other commands

| Command                                  | Description                             |
| ---------------------------------------- | --------------------------------------- |
| `dokku agent`                            | Show short usage                        |
| `dokku agent:help`                       | Full help                               |
| `dokku agent:run [config] [message...]`  | Run Docker Agent (TUI or with messages) |
| `dokku agent:exec <config> <message...>` | Run agent in headless mode              |

```bash
# Passthrough examples (API keys from plugin are loaded automatically)
dokku agent:run agent.yaml "Fix the bug in auth.go"
dokku agent:exec agent.yaml "Create a Dockerfile for a Python Flask app"
```

## Linking and AGENT_API_URL

When you link an Agent API service to an app, the plugin sets `AGENT_API_URL` to the service’s base URL (e.g. `http://172.17.0.1:31415`). App containers can call the [Docker Agent API](https://docker.github.io/docker-agent/features/api-server/) (e.g. `POST /api/sessions`, `POST /api/sessions/:id/agent/root`) using this URL.

The default host (`172.17.0.1`) is the Docker bridge gateway so containers can reach the host. If your setup differs, set `AGENT_API_DEFAULT_HOST` before linking (e.g. in the environment of the user that runs `dokku`).

## License

MIT
