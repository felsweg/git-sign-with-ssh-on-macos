# Working with Yubikey for SSH Auth and Signing on MacOS

_MacOS comes bundled with OpenSSH without the ability to create resident keys on a yubikey. In order for yubikey to be loaded via the ssh-agent, we require nix to be installed on the base system_

## Environment

Assuming `nix` is installed, you can run

```
user@system> nix-shell -p yubikey-manager openssh
```

to start an interactive nix shell with `yubikey-manager` and `openssh` readily available. We need `ssh-agent` to be loaded.

```
user@system-nix>eval $(ssh-agent -s)
```

## Creating Keys

We are going to create two types of keys
- A signing key with `ed25519` 
- An auth key with `ecdsa` 


### Signing key with ed25519

`ssh-keygen` will create token handles for the key stored inside the yubikey. Those handles can safely be deleted.

``` 
ssh-keygen -O resident -O application=ssh:github-felsweg-sign-02 -C "github-felsweg-sign-02" -t ed25519-sk -q -N '' -f /Users/thm/.ssh/id_github_felsweg_sign_02
```

### Auth key with ECDSA

```
ssh-keygen -O resident -O application=ssh:github-felsweg-auth-02 -C "github-felsweg-auth-02" -t ecdsa-sk -q -N '' -f /Users/thm/.ssh/id_github_felsweg_auth_02
```

## Loading Keys

Both keys are created inside the yubikey as `resident` keys, and can be loaded via 
```
user@system-nix> ssh-add -K
``` 

`ssh-add` will prompt you for the PIN for the yubikey (see `yubikey-manager` manual on how to properly set the PIN). 

## Showing Keys

If both keys are loaded successfully, you can list the keys and display the public key via 

```
user@system-nix> ssh-add -L
```

This output something similar to this list:
```
sk-ecdsa-sha2-nistp256@openssh.com <public key>

sk-ssh-ed25519@openssh.com <public key>

```

## Testing

After placing both keys successfully in GitHub at your profile, we can now test the connection to GitHub via 

```
user@system-nix> ssh -T git@github.com
```

A short message should indicate the success of the connection
```
Hi <username>! You've successfully authenticated, but GitHub does not provide shell access.
```

## Setting up Git for using the SSH Signing key

Git can use the previously generated SSH key to sign messages.

```
git config --global user.name "User Name"
git config --global user.email "user.name@email.com"
git config --global commit.gpgsign true
git config --global gpg.format ssh
git config --global user.signingKey "key::sk-ssh-ecdsa <PUBLIC_KEY>"
```

Where `<PUBLIC_KEY>` is the public part of your loaded signing key (again, list via `ssh-add -L` to view your public keys ) 

If you used the `--global` flag your `~/.gitconfig` should look like this

```
[user]
        name = User Name
        email = user.name@email.com
        signingKey = key::sk-ssh-ed25519@openssh.com <PUBLIC_KEY>
[commit]
        gpgsign = true
[gpg]
        format = ssh
[gpg "ssh"]
        allowedSignersFile = /Users/thm/.ssh/allowed_signers
```


Create a file called `~/.ssh/allowed_signers` with the following contents

```
user.name@email.com sk-ssh-ed25519@openssh.com <PUBLIC_KEY>
```

### Testing Signing with the SSH Key in Git

Create an empty project, run `git init` in it to create a git repo and run

```
user@system-nix> git commit --allow-empty -m "this empty commit should be signed"
```

Then after committed successfully, you can verify the signature.

```
user@system-nix> git log --show-signature
```
