# Slack to Codex CLI bridge

A small Linux service for an always-on host. It listens for `@mentions` in approved Slack channels, runs `codex exec` inside one configured project directory, and replies in the Slack thread. It uses Slack Socket Mode, so it does not need a public inbound server.

## Access limits

- Only Slack user IDs listed in `ALLOWED_USER_IDS` can start work.
- Only channel IDs listed in `ALLOWED_CHANNEL_IDS` are accepted.
- The Codex process starts in the single `CODEX_WORKDIR` directory with the `workspace-write` sandbox.
- Jobs run one at a time. Each task has a 10 minute time limit.
- The bot only handles channel mentions; it does not read DMs or monitor channel history.

Codex can edit files and run commands allowed by its sandbox in the configured project. Use a dedicated project checkout on the Orange Pi, preferably with Git, and keep Slack access limited to people you trust to request those changes.

## 1. Create the Slack app

Create an internal Slack app for your workspace using `slack-app-manifest.json`. Create an app-level token with `connections:write`, then install the app. Copy the bot token, app-level token, allowed Slack channel ID, and your Slack member ID into the private `.env` on the Orange Pi. Never commit or send the tokens in chat.

## 2. Install on the Orange Pi

Copy this directory to the Orange Pi, then create the dedicated empty workspace and install the bridge:

```sh
mkdir -p /home/orangepi/projects/slack-codex-workspace
cd slack-codex-agent
python3 -m venv .venv
. .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env
chmod 600 .env
```

Edit `.env` with the two Slack tokens, your Slack user ID, and the channel ID. The configured workspace is `/home/orangepi/projects/slack-codex-workspace`. Install and sign in to Codex CLI as the Linux service account that will run this bridge. Confirm `codex` is on that account's `PATH` and that the account can access the workspace directory.

Start it once in the foreground with `.venv/bin/python app.py`. In Slack, mention the bot in the allowed channel with a small, reversible task. The bot posts a queued notice and replies with Codex's final response in that thread.

## 3. Keep it running with systemd

Copy `slack-codex-agent.service.example` to `/etc/systemd/system/slack-codex-agent.service`, update `User`, `WorkingDirectory`, and `EnvironmentFile`, then run:

```sh
sudo systemctl daemon-reload
sudo systemctl enable --now slack-codex-agent
sudo systemctl status slack-codex-agent
```

The service example restarts after process failure. Its logs contain service status and errors, not task text or Slack tokens.

## Notes

The Orange Pi needs outbound internet access to Slack and the Codex service. Codex authentication and Slack app approval are separate setup steps. The bridge does not expose the Orange Pi to inbound internet connections.
