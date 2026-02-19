# SSH + GitHub Agent-Forwarding Tech Note

## Goal

Enable git clone on a remote host without copying private SSH keys there, while keeping
a dedicated GitHub machine user for repo access.

## 1) GitHub side (dedicated machine user)

- Create/use dedicated machine user in GitHub (example: relux-git-bot).
- Add `~/.ssh/relux-git-bot.pub` to that user’s SSH keys.
- Add the user to the org (or project team) with least-privilege access:
  - repo read (or required scope), no more than needed.
- If org uses SSO, authorize the key/session for the org.

## 2) Key storage (local vs remote)

- Local machine:
  - Private: `~/.ssh/relux-git-bot` (mode 600)
  - Public: `~/.ssh/relux-git-bot.pub` (mode 644)
- Remote machine:
  - **No private key file** for this flow. Uses forwarded agent socket from local during SSH session.
  - Public: `~/.ssh/relux-git-bot.pub` (mode 644). *Note: Copying the public key here allows us to filter identities from the forwarded agent.*

## 3) Local SSH Config

Keep the default `github.com` host untouched for personal use; use an explicit alias for the bot locally.

`~/.ssh/config` (local):

```bash
Host github-relux
    HostName github.com
    User git
    IdentityFile ~/.ssh/relux-git-bot
    AddKeysToAgent yes
    IdentitiesOnly yes
    UseKeychain yes # macOS specific

Host relux
    HostName <remote-hostname-or-ip>
    User <remote-ssh-user>
    ForwardAgent yes
```

## 4) Persist agent across local sessions (macOS)

In local shell startup (`~/.zshrc`), ensure the agent is available and the key is loaded:

```bash
if [ -z "$SSH_AUTH_SOCK" ] || [ ! -S "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" >/dev/null 2>&1
fi
# Modern macOS uses --apple-use-keychain (older versions used -K)
ssh-add --apple-use-keychain ~/.ssh/relux-git-bot >/dev/null 2>&1
```

Reload shell (`source ~/.zshrc`) and verify once locally:

```bash
ssh-add -l | grep -F "relux-git-bot"
```

## 5) Remote SSH Config (Identity Filtering)

Because agent forwarding exposes *all* your loaded local keys to the remote machine, GitHub might authenticate you with your personal key instead of the bot key. To prevent this, set up a config on the **remote** machine using the public key to filter the correct identity from the forwarded agent.

`~/.ssh/config` (remote):

```bash
Host github-relux
    HostName github.com
    User git
    # Pointing to the PUBLIC key forces SSH to use the matching private key from the forwarded agent
    IdentityFile ~/.ssh/relux-git-bot.pub
    IdentitiesOnly yes
```

## 6) Remote workflow

From local, connect with the agent forwarded (configured in step 3):

```bash
ssh relux
```

On remote, verify the agent forwarded successfully:

```bash
echo "$SSH_AUTH_SOCK"
ssh-add -l | grep -F "relux-git-bot"
```

*(If `ssh-add -l` on remote does not show the key, disconnect and reconnect explicitly with `ssh -A relux`)*

On remote, test authentication using the alias defined in the remote config:

```bash
ssh -T git@github-relux
# Expected output: "Hi relux-git-bot! You've successfully authenticated..."
```

On remote, clone using the alias:

```bash
git clone git@github-relux:relux-works/skill-project-management.git
```

## 7) Why this avoids prompts

- Passphrase is only asked once on local when loading the key into the local agent (or handled silently by macOS Keychain).
- Remote sessions reuse the local agent over forwarding.
- No remote private key prompts, since no private key is stored remotely.
- Multiple keys won't collide because the remote SSH config uses the public key to filter the exact identity needed from the agent.
