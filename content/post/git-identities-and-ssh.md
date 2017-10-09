+++
title = "Git Identities and Ssh"
date = 2017-10-09T11:22:39+02:00
draft = false
#notonhomepage = true
highlight_languages = [ "yaml" ]

# Tags and categories
# For example, use `tags = []` for no tags, or the form `tags = ["A Tag", "Another Tag"]` for one or more tags.
tags = [ "git", "identities", "ssh", "config" ]
categories = [ "Tools" ]

# Featured image
# Place your image in the `static/img/` folder and reference its filename below, e.g. `image = "example.jpg"`.
[header]
image = "headers/git-identities-and-ssh.png"
caption = "[:information_source:](# 'Copyright Derek Hass, intended as fair use')"

+++

If you sometimes work on both personal and professional projects that are hosted
on GitHub, chances are that you'd like to separate identities. Maybe you have
several accounts, maybe it's just a matter of making the commits use different
emails.

In this post, we'll see how to manage such separate identities, at the SSH level
and the Git level.

<!--more-->

# Configuring Per-Repository Author Information in Git

{{% alert danger %}}
This requires Git 2.13 or above
{{% /alert %}}

The idea is to organize cloned repositories in a hierarchy of folders, which
will automatically get the correct configuration thanks to the [Conditional
Includes](https://git-scm.com/docs/git-config#_conditional_includes) feature
from `Git 2.13`.

Thanks to that feature, one can alter the configuration depending to a pattern,
e.g. on the absolute path of a given repo. That can be put to use to alter the
authorship information for instance.

Define such a pattern inside the `.gitconfig` file in your home folder:

```ini
[user]
    name = Bruce Wayne
    email = <bruce.wayne@wayneindustries.com>

[includeIf "gitdir:~/BatCave/"]
    path = .gitconfig-secret
```

Then define the overriding configuration of the `.gitconfig-secret`:

```ini
[user]
    name = Batman
    email = <caped.crusader@justiceleague.com>
```

Now, every time you clone a Git repository under the `BatCave` base folder, Git
will use your secret batman identity for your commits :sunglasses: :+1:

{{% alert note %}}
This local configuration is visible with a `git config --list` **IF** you're
actually inside a git-enabled folder.
{{% /alert %}}

# Configuring Different GitHub Accounts Via SSH Host Configuration
The next step is when you have different GitHub accounts (e.g. your work requires
you to have a dedicated account, and you have personal pet projects that you
manage using a different account).

Chances are, you'd like to authenticate to these separate accounts using SSH.
For this, you need a little bit of setup, both for SSH and Git.

We'll assume that you have generated two SSH keys: `work` and `personal`, linked
to the adequate GitHub accounts.

First let's setup SSH with _Host configuration_. In your `~/.ssh/` folder,
create a `config` file (if it doesn't already exists).

In there, put domain-specific configurations like these two:

```yaml
#work SSH identity (default)
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_github

#personal SSH identity
Host batman.github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_rsa_githubbatman
```

Let's dive into a bit more details on the second configuration:

 1. The `Host batman.github.com` is the key: SSH will recognize this "virtual"
 domain, e.g. in a `git remote` URL, and associate this configuration to it...
 2. ...but it will call the _actual_ `github.com` domain, as configured by the
 `HostName` entry.
 3. The user is still traditionally `git` in SSH git remotes
 4. The `IdentityFile` is the SSH key to use.

Next you still need to tell `git` to use that SSH identity. For this, you need to
**edit your git SSH remote(s)** in order to replace the `github.com` domain in the
URL with `batman.github.com`:

```
git remote set-url origin git@batman.github.com:batman/batcave-OS.git
```

Think of it as a kind of **DNS for Git** :smile: