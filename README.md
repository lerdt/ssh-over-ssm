# ssm-over-ssh
Connect to instances using SSM over SSH. No keys need to be stored locally or on the server.

## Info and requirements

Recently I was required to administer AWS instances via Session Manager. After downloading the required plugin and initiating a SSM session locally using `aws ssm start-session` I found myself in a situation where I couldn't quickly copy over a file from my machine to the server (e.g. SCP, sftp, rsync etc). After some reading of the AWS documentation I found that it's possible to establish a SSH connection over SSM, solving the issue of not being able to copy data to and from the server. What's also cool is that you can connect to private instances inside your VPC without a public-facing bastion and you don't need to store any SSH keys on the server. As long as a user has the required IAM access they can connect over SSH with this approach. I don't see an issue as long you're properly locking down IAM access and enforcing MFA.

### Requirements
- Instances must have access to ssm.{region}.amazonaws.com
- IAM instance profile allowing SSM access must be attached to EC2 instance
- SSM agent must be installed on EC2 instance
- AWS cli requires you install `session-manager-plugin` locally

Existing instances with SSM agent already installed may require you to update the agent.

### How it works
You configure each of your instances in your SSH config and specify `ssm-over-ssh.sh` to be executed as a ProxyCommand with your `AWS_PROFILE` environment variable set.
If your key is available via ssh-agent it will be used by the script, otherwise a temporary key will be created, used and destroyed on termination of the script. The public key is copied across to the instance using `aws send-command` and removed after 15 seconds, allowing you enough time to authenticate via SSH.


## Installation and Usage

This tool is intended to be used in conjunction with `ssh`. It requires that you've configured your awscli config (`~/.aws/{config,credentials}`) and you spend a small amount of time planning and updating your ssh config.

### Listing and updating SSM instances

First, we need to make sure the agent on each of our instances is up-to-date. You can use `aws ssm describe-instance-information` to list instances and `aws ssm send-command` to update them. Alternatively, I've included a small python script to quickly list or update your instances:

Check your instances
```
[elpy@testbox ~]$ AWS_PROFILE=int-monitor1 python3 ssm_instances.py
instance id           |ip                    |agent up-to-date      |platform              |name
------------------------------------------------------------------------------------------------------------------
i-0xxxxxxxxxxxxx3b4   |10.xx.xx.6            |False                 |Ubuntu                |instance1
i-0xxxxxxxxxxxxx76e   |10.xx.xx.142          |False                 |Ubuntu                |instance2
i-0xxxxxxxxxxxxx1b6   |10.xx.xx.75           |False                 |Ubuntu                |instance3
i-0xxxxxxxxxxxxxac8   |10.xx.xx.240          |False                 |Ubuntu                |instance4
i-0xxxxxxxxxxxxxb1a   |10.xx.xx.206          |False                 |Ubuntu                |instance5
i-0xxxxxxxxxxxxx504   |10.xx.xx.84           |False                 |Amazon Linux          |
i-0xxxxxxxxxxxxx73d   |10.xx.xx.48           |False                 |Ubuntu                |instance6
i-0xxxxxxxxxxxxxd56   |10.xx.xx.201          |False                 |Ubuntu                |instance7
i-0xxxxxxxxxxxxxfe9   |10.xx.xx.143          |False                 |CentOS Linux          |instance8
i-0xxxxxxxxxxxxxb8e   |10.xx.xx.195          |False                 |Ubuntu                |instance9

```

Update all instances
```
[elpy@testbox ~]$ AWS_PROFILE=int-monitor1 python3 ssm_instances.py update
success
```

### SSH config

Now that all of our instances are running an up-to-date agent we need to update our SSH config for seamless SSH access.

Example of basic `~/.ssh/config`:
```
Host confluence-prod.personal
  Hostname i-0xxxxxxxxxxxxxe28
  User ec2-user
  ProxyCommand bash -c "AWS_PROFILE=atlassian-prod ~/bin/ssm-over-ssh.sh %h %r"
  IdentityFile ~/.ssh/ssm-ssh-tmp
  PasswordAuthentication no
  GSSAPIAuthentication no

Host confluence-stg.personal
  Hostname i-0xxxxxxxxxxxxxe49
  User ec2-user
  ProxyCommand bash -c "AWS_PROFILE=atlassian-nonprod ~/bin/ssm-over-ssh.sh %h %r"
  IdentityFile ~/.ssh/ssm-ssh-tmp
  PasswordAuthentication no
  GSSAPIAuthentication no

Host bitbucket-prod.personal
  Hostname i-0xxxxxxxxxxxxx835
  User ec2-user
  ProxyCommand bash -c "AWS_PROFILE=atlassian-prod ~/bin/ssm-over-ssh.sh %h %r"
  IdentityFile ~/.ssh/ssm-ssh-tmp
  PasswordAuthentication no
  GSSAPIAuthentication no
```
Above we've configured 3 separate instances for SSH access with a simple host to remember. If you only have 1 or 2 instances to setup this might be acceptable but when you've got a large number of different AWS profiles (think: work-internal, work-clients, personal) there's quite a bit of repetition so we can take a different approach.

I prefer to use includes and I've set mine up similar to below.

Example `~/.ssh/config`:
```
Host *
  Include conf.d/internal/*
  Include conf.d/clients/*
  Include conf.d/personal/*
  KeepAlive yes
  Protocol 2
  ServerAliveInterval 20
  ConnectTimeout 10

Match exec "find ~/.ssh/conf.d -type f -name '*_ssm' -exec grep '%h' {} +"
  IdentityFile ~/.ssh/ssm-ssh-tmp
  PasswordAuthentication no
  GSSAPIAuthentication no
```

Example `~/.ssh/conf.d/personal/atlassian-prod_ssm`:
```
Host confluence-prod.personal
  Hostname i-0xxxxxxxxxxxxxe28

Host confluence-stg.personal
  Hostname i-0xxxxxxxxxxxxxe49

Host bitbucket-prod.personal
  Hostname i-0xxxxxxxxxxxxx835

Match host i-*
  User ec2-user
  ProxyCommand bash -c "AWS_PROFILE=atlassian-prod ~/bin/ssm-over-ssh.sh %h %r"
```

With the above approach I separate my ssh config into fragments. All SSM hosts are saved in a fragment ending in '_ssm'. Within the config fragment I include each instance, their corresponding hostname (instance ID) and a Match directive containing the relevant User and ProxyCommand.

Once again, if you don't have many accounts/instances to work with you could simplify this by simply having `~/.ssh/config` and `~/.ssh/config_ssm` or similar.

### Testing/debugging SSH config

Show which config file and Host you match against and the final command executed by SSH:
```
ssh -G confluence-prod.personal 
```

Debug connection issues:
```
ssh -vvv user@host
```

Once you've tested it and you're confident it's all correct give it a go! Remember to place `ssm-over-ssh.sh` in `~/bin/` (or wherever you prefer).

SSH:
```
[elpy1@testbox ~]$ aws-mfa
INFO - Validating credentials for profile: default
INFO - Your credentials are still valid for 14105.807801 seconds they will expire at 2020-01-25 18:06:08
[elpy1@testbox ~]$ ssh confluence-prod.personal
Last login: Sat Jan 25 08:59:40 2020 from localhost

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-10-xx-x-x06 ~]$ logout
Connection to i-0fxxxxxxxxxxxxe28 closed.
```

SCP:
```
[elpy@testbox ~]$ scp ~/bin/ssm-over-ssh.sh bitbucket-prod.personal:~
ssm-over-ssh.sh                                                                                       100%  366    49.4KB/s   00:00

[elpy@testbox ~]$ ssh bitbucket-prod.personal ls -la ssm\*
-rwxrwxr-x 1 ec2-user ec2-user 366 Jan 26 07:27 ssm-over-ssh.sh
```
