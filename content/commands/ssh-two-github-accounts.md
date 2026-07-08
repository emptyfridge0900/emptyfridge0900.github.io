+++
title = "Using two GitHub accounts with SSH on one machine (work + personal)"
date = 2026-06-17

[taxonomies]
categories = ["post"]
tags = ["git", "ssh", "github", "command"]
+++

If you have two GitHub accounts, one for work and one personal, using both on the same Mac is awkward with HTTPS.
macOS Keychain stores one token per `github.com`, so the token from the account you logged into most recently can overwrite the previous one.

```
remote: Permission to emptyfridge0900/repo.git denied to bds0900.
fatal: unable to access '...': The requested URL returned error: 403
```

The clean fix is to create separate SSH keys per account and route them through `~/.ssh/config`.

---

## Overall flow

```
1. Check existing SSH keys
2. Understand SSH key file naming
3. Create a new SSH key for the work account (ssh-keygen)
4. Register Host aliases in ~/.ssh/config
5. Add keys to the SSH agent (ssh-add)
6. Register the public key in the work GitHub account
7. Test the connection (ssh -T)
8. Change existing repo remote URLs to SSH
9. Make the setup survive reboot (AddKeysToAgent)
10. Separate GitHub CLI(`gh`) accounts too (`GH_CONFIG_DIR`)
```

---

## 1. Check existing SSH keys

```bash
ls -la ~/.ssh/
```

If `id_ed25519` and `id_ed25519.pub` exist, you already have one SSH key.
To see which account it belongs to, check the last part of the public key file.

```bash
cat ~/.ssh/id_ed25519.pub
# ssh-ed25519 AAAA... personal@gmail.com
```

Also verify that it connects to GitHub.

```bash
ssh -T git@github.com
# Hi emptyfridge0900! You've successfully authenticated...
```

`ssh -T` tests authentication without opening a terminal shell session (`-T` = no pseudo-terminal).

---

## 2. Understand SSH key file naming

An SSH key filename has two parts.

| Part | Meaning |
|---|---|
| `id_` | Short for "identity." A standard prefix for files used by SSH authentication. |
| `ed25519` | The encryption algorithm name, Edwards-curve Digital Signature Algorithm 25519. It is currently the recommended default. |

Creating a key always produces two files.

- `id_ed25519` — **private key**. Never share this.
- `id_ed25519.pub` — **public key**. This is the file you register in GitHub.

---

## 3. Create an SSH key for the work account

```bash
ssh-keygen -t ed25519 -C "work@company.com" -f ~/.ssh/id_ed25519_company -N ""
```

Option meanings:

| Option | Meaning |
|---|---|
| `-t ed25519` | Sets the algorithm type. `ed25519` is the current standard. It is shorter, faster, and safer than older `rsa` usage. |
| `-C "work@company.com"` | Comment. A label appended to the end of the public key file. It helps identify which account the key belongs to. It does not have to be an email, but that is common. |
| `-f ~/.ssh/id_ed25519_company` | Sets the output path. The default is `~/.ssh/id_ed25519`, but the personal key already uses that name, so this key needs a different filename. |
| `-N ""` | Sets an empty passphrase. The command creates the key without opening an interactive prompt. |

After running it, two files are created.

```
~/.ssh/id_ed25519_company      # private key
~/.ssh/id_ed25519_company.pub  # public key
```

---

## 4. Configure `~/.ssh/config`

The SSH client reads `~/.ssh/config` when connecting and decides which key to use.
Create a `Host` alias per account.

```
# Personal GitHub (emptyfridge0900)
Host github-personal
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  AddKeysToAgent yes

# Work GitHub (bds0900)
Host github-company
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519_company
  AddKeysToAgent yes
```

Directive meanings:

| Directive | Meaning |
|---|---|
| `Host github-personal` | The alias for this block. When you use `git@github-personal:...`, this block is applied. |
| `HostName github.com` | The real server address. Even with different aliases, both connect to `github.com`. |
| `User git` | SSH username. GitHub always uses the `git` user; account identity is determined by the key. |
| `IdentityFile` | Private key path to use for this alias. |
| `AddKeysToAgent yes` | Automatically adds the key to the SSH agent on first use, so you do not have to run `ssh-add` manually after reboot. |

---

## 5. Add keys to the SSH agent

```bash
ssh-add ~/.ssh/id_ed25519
ssh-add ~/.ssh/id_ed25519_company
```

The **SSH agent(`ssh-agent`)** is a background process that keeps private keys in memory and provides them when needed.
Once a key is registered in the agent, you do not have to specify the key file path every time.

Main `ssh-add` options:

| Option | Meaning |
|---|---|
| `~/.ssh/id_ed25519` | Private key file path to add. |
| `-l` | List keys currently registered in the agent. |
| `-d ~/.ssh/id_ed25519` | Remove a specific key from the agent. |
| `-D` | Remove all keys from the agent. |

Verify registration:

```bash
ssh-add -l
# 256 SHA256:xxx personal@gmail.com (ED25519)
# 256 SHA256:yyy work@company.com (ED25519)
```

> Keys added with `ssh-add` disappear after reboot. If `AddKeysToAgent yes` is in the config, SSH automatically re-adds the key on first use.

---

## 6. Register the public key in the work GitHub account

Copy the public key to the clipboard.

```bash
cat ~/.ssh/id_ed25519_company.pub | pbcopy
```

In GitHub, go to **Settings** -> **SSH and GPG keys** -> **New SSH key**, paste it, and save.

---

## 7. Test connections

```bash
ssh -T git@github-personal
# Hi emptyfridge0900! You've successfully authenticated...

ssh -T git@github-company
# Hi bds0900! You've successfully authenticated...
```

`github-personal` and `github-company` are aliases from `~/.ssh/config`.
They both connect to `github.com`, but they use different keys.

---

## 8. Change remote URLs

### Existing repo (HTTPS -> SSH)

```bash
git remote set-url origin git@github-personal:emptyfridge0900/repo-name.git
```

Check with `git remote -v`:

```
origin  git@github-personal:emptyfridge0900/repo-name.git (fetch)
origin  git@github-personal:emptyfridge0900/repo-name.git (push)
```

### Work repo

```bash
git remote set-url origin git@github-company:bds0900/repo-name.git
```

### Cloning a new repo

```bash
# Personal account repo
git clone git@github-personal:emptyfridge0900/repo-name.git

# Work account repo
git clone git@github-company:bds0900/repo-name.git
```

URL structure: `git@<Host alias>:<GitHub username>/<repository name>.git`

---

## 9. Make it survive reboot

Keys added to the agent with `ssh-add` **disappear after reboot** because terminal session memory is cleared.

Putting `AddKeysToAgent yes` in `~/.ssh/config` solves this.

```
Host github-personal
  ...
  AddKeysToAgent yes   # <- this line

Host github-company
  ...
  AddKeysToAgent yes   # <- this line
```

What `AddKeysToAgent yes` does:

- After reboot, the first `git push` or `git pull` causes SSH to read `~/.ssh/config`.
- If the matching `Host` has `AddKeysToAgent yes`, SSH automatically runs the equivalent of `ssh-add` and loads the key into the agent.
- After that, it will not ask again in the same session.

So after reboot, you do not need to run `ssh-add` manually. The first git command reloads the key automatically.

> On macOS, the `~/.ssh/config` file itself is persistent. It remains after reboot.
> What disappears is **agent memory(the result of ssh-add)**, and `AddKeysToAgent yes` fills it again automatically.

---

## Final setup summary

| Account | Host alias | Key file |
|---|---|---|
| `emptyfridge0900` (personal) | `git@github-personal` | `~/.ssh/id_ed25519` |
| `bds0900` (work) | `git@github-company` | `~/.ssh/id_ed25519_company` |

Now you do not have to think about GitHub accounts when running `git push` or `git pull`.
The Host alias in the repo remote URL chooses the correct key.

---

## Extra

## 10. Separate GitHub CLI(`gh`) accounts too

At this point, `git push` and `git pull` are separated correctly by account-specific SSH keys.
But GitHub CLI commands such as `gh issue create` and `gh pr create` use GitHub API tokens, not SSH keys.
So even if the remote URL is `git@github-personal:...`, issue creation in a personal repo can fail if `gh` is logged in with the work account token.

The first solution that comes to mind is `gh auth switch`.

```bash
gh auth switch -u emptyfridge0900
gh issue create -R emptyfridge0900/repo-name

gh auth switch -u bds0900
gh issue create -R bds0900/repo-name
```

But switching accounts every time is annoying and easy to get wrong.
A better approach is to separate the `gh` config directory per account.
`gh` can change where it stores configuration through the `GH_CONFIG_DIR` environment variable.

Add aliases to a shell config file such as `~/.zshrc`.

```bash
alias ghp='GH_CONFIG_DIR=$HOME/.config/gh-personal gh'
alias ghc='GH_CONFIG_DIR=$HOME/.config/gh-company gh'
```

Then log in once with each alias.

```bash
GH_CONFIG_DIR=$HOME/.config/gh-personal gh auth login -h github.com --git-protocol ssh
GH_CONFIG_DIR=$HOME/.config/gh-company gh auth login -h github.com --git-protocol ssh
```

After that, distinguish accounts by command instead of switching accounts.

```bash
# Create an issue in a personal repo using the personal account
ghp issue create -R emptyfridge0900/repo-name

# Create an issue in a work repo using the work account
ghc issue create -R bds0900/repo-name
```

The final split looks like this:

| Tool | Account separation method | Example |
|---|---|---|
| `git` | SSH Host alias | `git@github-personal:emptyfridge0900/repo-name.git` |
| `gh` | Separate `GH_CONFIG_DIR` | `ghp issue create -R emptyfridge0900/repo-name` |

With this setup, both `git` and `gh` can be used without account switching.
Personal work stays fixed to `github-personal` + `ghp`, and work tasks stay fixed to `github-company` + `ghc`.
