# SSH + GitHub Agent-Forwarding Tech Note

## Goal

Enable git clone on a remote host without copying private SSH keys there, while keeping a
dedicated GitHub machine user for repo access.

## 1) GitHub side (dedicated machine user)

- Create/use dedicated machine user in GitHub (example: relux-git-bot).
- Add `~/.ssh/relux-git-bot.pub` to that user’s SSH keys.
- Add the user to the org (or project team) with least-privilege access:
  - repo read (or required scope), no more than needed.
- If org uses SSO, authorize the key/session for the org.

## 2) Key ownership (local vs remote)

- Local machine:
  - Private: `~/.ssh/relux-git-bot` (mode 600)
  - Public: `~/.ssh/relux-git-bot.pub` (mode 644)
- Remote machine:
  - **No private key file** for this flow.
  - Public: `~/.ssh/relux-git-bot.pub` (mode 644), required for remote key filtering.

## 3) Local setup

### 3.1 Local SSH aliases

Keep default `github.com` behavior untouched for personal work and use a named alias for
the bot:

`~/.ssh/config` (local):

```bash
Host github-relux
    HostName github.com
    User git
    IdentityFile ~/.ssh/relux-git-bot
    AddKeysToAgent yes
    IdentitiesOnly yes
    UseKeychain yes

Host relux
    HostName <remote-hostname-or-ip>
    User <remote-ssh-user>
    ForwardAgent yes
```

### 3.2 Keep agent persistent locally (macOS)

In local shell startup (`~/.zshrc`), ensure the agent is available and the key is loaded:

```bash
if [ -z "$SSH_AUTH_SOCK" ] || [ ! -S "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" >/dev/null 2>&1
fi
# Modern macOS uses --apple-use-keychain (older versions used -K)
ssh-add --apple-use-keychain ~/.ssh/relux-git-bot >/dev/null 2>&1
```

Reload shell (`source ~/.zshrc`) and verify locally:

```bash
ssh-add -l | grep -F "relux-git-bot"
```

## 4) Deploy public key to remote (required for identity filtering)

From your local machine, copy the bot public key and set safe permissions:

```bash
# create ssh directory and copy key
authenticated command
ssh relux 'mkdir -p ~/.ssh && chmod 700 ~/.ssh'
cat ~/.ssh/relux-git-bot.pub | ssh relux 'cat > ~/.ssh/relux-git-bot.pub'
ssh relux 'chmod 644 ~/.ssh/relux-git-bot.pub'
```

Verify:

```bash
ssh relux 'ls -l ~/.ssh/relux-git-bot.pub && sha256sum ~/.ssh/relux-git-bot.pub'
```

> You can replace the three-step deploy with `scp` if you prefer.

## 5) Remote setup

`~/.ssh/config` (remote) to force SSH to use only the bot identity from forwarded agent:

```bash
Host github-relux
    HostName github.com
    User git
    # Pointing to the PUBLIC key makes SSH pick the matching private key from agent
    IdentityFile ~/.ssh/relux-git-bot.pub
    IdentitiesOnly yes
```

## 6) Remote workflow

From local:

```bash
ssh relux
```

On remote, verify the forwarded agent contains the key:

```bash
echo "$SSH_AUTH_SOCK"
ssh-add -l | grep -F "relux-git-bot"
```

If `ssh-add -l` on remote does not show the key, reconnect with forced forwarding:

```bash
ssh -A relux
```

Test bot auth and clone using the remote alias:

```bash
ssh -T git@github-relux
# Expected output: "Hi relux-git-bot! You've successfully authenticated..."

git clone git@github-relux:relux-works/skill-project-management.git
```

## 7) Why this avoids prompts

- Passphrase is asked once on local when loading the key into local agent (or handled silently by keychain).
- Remote sessions reuse the local agent over forwarding.
- No private key prompts on remote, since no private key file is stored there.
- Multiple local keys won’t collide because remote alias uses `IdentitiesOnly` + public-key filtering.
