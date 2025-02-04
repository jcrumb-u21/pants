---
title: "GitHub Actions macOS ARM64 runners"
slug: "ci-for-macos-on-arm64"
hidden: false
createdAt: "2022-06-05T15:31:27.665Z"
updatedAt: "2022-06-12T08:38:54.894Z"
---
Apple is phasing out their X86_64 hardware, and all new macOS systems are based on the M1 ARM64 processor. Pants must run on these systems, which means we need an M1 CI machine on which to test and package Pants.

Unfortunately, GitHub Actions does not yet have hosted runners for MacOS ARM64. So we must run our own self-hosted runner. This document describes how to set one up. It is intended primarily for Pants maintainers who have to maintain our CI infrastructure, but since there is not much information online about how to set up self-hosted runners on M1, it may be useful as a reference to other projects as well.  One useful resource we did find is [this blog post](https://betterprogramming.pub/run-github-actions-self-hosted-macos-runners-on-apple-m1-mac-b559acd6d783) by Soumya Mahunt, so our thanks to them.

If you find any errors or omissions in this page, please let us know on [Slack](doc:getting-help#slack) or provide corrections via the "Suggest Edits" link above. 

The machine
-----------

As yet there aren't many options for a hosted M1 system:

- AWS has a [preview program](https://aws.amazon.com/about-aws/whats-new/2021/12/amazon-ec2-m1-mac-instances-macos/), which you can sign up for and hope to get into. Once these instances are generally available we can evaluate them as a solution.
- You can buy an M1 machine and stick it in a closet. You take on the risk of compromising your  
  network if the machine is compromised by a rogue CI job. 
- You can rent a cloud-hosted M1 machine by the month from [MacStadium](https://www.macstadium.com/).

We've gone with the MacStadium approach for now.

Connecting to the machine
-------------------------

Since this is machine is [a pet, not cattle](https://iamondemand.com/blog/devops-concepts-pets-vs-cattle/), we allow ourselves a somewhat manual, bespoke setup process (we can script this up if it becomes necessary). There are two ways to connect to the machine:

- Via VNC remote desktop from another macOS machine (not necessarily an M1)
- Via SSH

In both cases, the first few setup steps will be done as the user `administrator` and the initial password for that user is provided by MacStadium. Once we create a role user, the subsequent steps will be run as that user.

### SSH

```shell Shell
$ ssh administrator@XXX.XXX.XXX.XXX
(administrator@XXX.XXX.XXX.XXX) Password:
%
```

### VNC

Enter `vnc://XXX.XXX.XXX.XXX` on the local machine's Safari address bar, substituting the machine's IP address, as given to you by MacStadium. Safari will prompt you to allow it to open the Screen Sharing app.

Screen Sharing will give you a login prompt. Once logged in, you can control the remote machine's desktop in the Screen Sharing window, and even share the clipboard across the two machines.

In this mode you can use the remote machine's desktop UI to make changes, or you can open a terminal and issue the same commands you would via SSH. 

A few of the steps below will have both SSH and VNC options, others only SSH (or terminal window in a remote desktop), or only VNC.

Setting up the machine
----------------------

### Change the initial password

The first step is to change the initial `administrator` password to something secure, since the initial password appears as cleartext in the MacStadium ticket 

#### SSH

```shell
# Will prompt for both the new and old passwords
% dscl . -passwd /Users/administrator
```

#### VNC

Go to  > System Preferences > Users & Groups, select the administrator user, click "Change Password..." and select a strong password.

### Ensure smooth restarts

#### SSH

```shell Shell
# Ensure that this shows a value of 1
% pmset -g | grep autorestart
# If it does not, run this
% sudo pmset -a autorestart 1
```

#### VNC

Go to  > System Preferences > Energy Saver and ensure that Restart After Power Failure is checked.

### Install software

Perform the following setup steps as `administrator`, some steps may request your password:

```Text Shell
# Install Rosetta 2, will prompt to accept a license agreement
% softwareupdate --install-rosetta

# Install XCode command-line tools
# IMPORTANT: This pops up a license agreement window on the desktop,
#  so you must use VNC to accept the license and complete the installation.
% xcode-select --install

# Install Homebrew
% /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
% echo 'eval "$(/opt/homebrew/bin/brew shellenv)"' >> /Users/administrator/.zshenv   
% eval "$(/opt/homebrew/bin/brew shellenv)"

# Install pyenv
% brew install pyenv

# Set up pyenv
% echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshenv
% echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshenv    
% echo 'eval "$(pyenv init -)"' >> ~/.zshenv
% source ~/.zshenv

# Install the AWS CLI
% brew install awscli



```

### Create a role user

We don't want to run actions as the administrator user, so we create a role account.

#### SSH

```shell
# Will prompt for password
% sudo sysadminctl  -addUser gha -fullName "GitHub Actions Runner" -password -

# Allow ssh'ing as gha
% sudo dseditgroup -o edit -a gha -t user com.apple.access_ssh
```

#### VNC

Go to  > System Preferences > Users & Groups and create a Standard account with the full name `GitHub Actions Runner`, the account name `gha` and a strong password. 

### Set up auto-login

This must be done from the remote desktop, via VNC, as `administrator`. 

Go to  > System Preferences > Users & Groups, and click the lock to make changes.

Click on Login Options and for Automatic login choose Github Actions Runner. Enter the `gha` user's password when prompted.

### Set up the role user

Perform the following setup steps after SSHing in as the `gha` role user:

```
# Set up Homebrew
% echo 'export PATH=$PATH:/opt/homebrew/bin/' >> ~/.zshenv   
...
# Set up pyenv
% echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshenv
% echo 'command -v pyenv >/dev/null || export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshenv    
% echo 'eval "$(pyenv init -)"' >> ~/.zshenv
% source ~/.zshenv
...
# Install Python 3.9
% pyenv install 3.9.13
% pyenv global 3.9.13
...
# Install rustup
% curl https://sh.rustup.rs -sSf | sh -s -- -y
```

Note that we use `.zshenv` because the runner will not execute in an interactive shell. 

Setting up the self-hosted runner
---------------------------------

### Installing the runner

On the GitHub repo's page, go to [Settings > Actions > Runners](https://github.com/pantsbuild/pants/settings/actions/runners).

Click "New self-hosted runner", select macOS and run all the Download and Configure commands it displays, as `gha`, on the remote machine. Set the labels to [`self-hosted`, `macOS`, `ARM64`, `macOS11`]. 

Accept the default values for other settings.

**Note:** The ARM64 GitHub Actions runner binary is still in pre-release status. If you don't want to rely on it, you can use the stable X86_64 binary under Rosetta. However in this case its subprocesses will run in X86_64 mode by default as well. So CI processes that care about platform (such as those that build and package native code) must be invoked with the `arch -arm64` prefix. Note that in this case GHA will always set the `X64` label on the runner, so be careful not to use that label for runner selection in your workflows if you also have X86_64 self-hosted runners.

### Runner setup

As `gha`, run:

```
% cd actions-runner

# Ensure that the runner starts when the machine starts.
% ./svc.sh install

# Set up some env vars the runner requires. 
% echo 'ImageOS=macos11' >> .env
% echo "XCODE_11_DEVELOPER_DIR=$(xcode-select -p)" >> .env
```

Testing it all out
------------------

Now use the MacStadium web UI to restart the machine. Once it comes back up it  
should be able to pick up any job with this setting:

```
    runs-on:
      - self-hosted
      - macOS11
      - ARM64
```
