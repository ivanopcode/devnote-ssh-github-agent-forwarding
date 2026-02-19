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

- Local machine only:
  - Private: `~/.ssh/relux-git-bot` (mode 600)
  - Public: `~/.ssh/relux-git-bot.pub` (mode 644)
- Remote machine:
  - No private key file for this flow.
  - Uses forwarded agent socket from local during SSH session.

## 3) Keep github.com host untouched; use explicit aliases

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

## 4) Persist agent across local sessions (macOS)

In local shell startup (`~/.zshrc`), ensure agent is available and key is loaded:

```bash
if [ -z "$SSH_AUTH_SOCK" ] || [ ! -S "$SSH_AUTH_SOCK" ]; then
  eval "$(ssh-agent -s)" >/dev/null 2>&1
fi
ssh-add --apple-use-keychain ~/.ssh/relux-git-bot >/dev/null 2>&1
```

Reload shell (`source ~/.zshrc`) and verify once locally:

```bash
ssh-add -l | grep -F "relux-git-bot"
```

## 5) Remote workflow

From local:

```bash
ssh relux
```

On remote:

```bash
echo "$SSH_AUTH_SOCK"
ssh-add -l | grep -F "relux-git-bot"
ssh -T git@github.com
git clone git@github.com:relux-works/skill-project-management.git
```

If `ssh-add -l` on remote does not show the key, reconnect with forwarding:

```bash
ssh -A relux
```

## 6) Why this avoids prompts

- Passphrase is only asked once on local when loading key into local agent.
- Remote sessions reuse local agent over forwarding.
- No remote private key prompts, since no private key is stored remotely.
