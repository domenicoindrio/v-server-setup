# v-server-setup

This document provides a step by step guide on how to set up a V-server with SSH-Key authentication, install and configure Nginx, and set up Git and Git access through SSH-Keys.

This setup was performed on a host maschine running MacOS Sequoia 15.3.2 through the Visual Studio Code zsh Terminal and a remote machine running Ubuntu 22.04.5 LTS (GNU/Linux 5.15.0-134-generic x86_64)

# Table of Contents

## Main Objectives
1. [SSH Login through SSH-Keys exclusively](#ssh-login-through-ssh-keys-exclusively)
2. [Nginx installation and configuration with a custom index](#nginx-installation-and-configuration-with-a-custom-index)
3. [Git configuration on the V-server](#git-configuration-on-the-v-server)

## Extra
- [Workflow improvements](#workflow-improvements)

# SSH Login through SSH-Keys exclusively
---
## Step 1: Create an SSH Key pair on local host 
Generate an SSH-Key pair on your local host through following command:

    $ ssh-keygen -t ed25519 -C "Comment"

> **[! Info]**  
> `-t` specifies the type of key (`rsa`, `ed25519`, etc.)  
> `-C` adds a comment at the end of the key (useful to keep track of the key purpose)

Provide a name and a location for the new key pair, or press enter to leave it default.  
Enter a Passphrase for an extra security layer, or leave it blank.  
Afterward, the Key pair is successfully generated.

The Terminal output for this process will look like this:

    $ ssh-keygen -t ed25519 -C "Comment"
    Generating public/private ed25519 key pair.
    Enter file in which to save the key (/home/username/.ssh/id_ed25519): 
    Enter passphrase (empty for no passphrase):
    Your identification has been saved in /home/username/.ssh/id_ed25519
    Your public key has been saved in /home/username/.ssh/id_ed25519.pub
    The key fingerprint is: 
    SHA256:<public key> Comment
    The key's randomart image is:
    +--[ED25519 256]--+
    |                 |
    |     o.          |
    |    .o+          |
    |   .o.=.         |
    |    o+=o.        |
    |     *o. +       |
    |    = o o o      |
    |   o O o .       |
    |    +B=.+.       |
    +----[SHA256]-----+

## Step 2: Copy the SSH Public Key to the server
In order to perform SSH-Key authentication on the server, the SSH public key needs to be copied on the server.
Here is the command:

    $ ssh-copy-id -i ~/.ssh/key.pub username@server_ipaddress

> **[! Info]**  
>`-i` specifies the key to use

A verbose log is displayed but the important info is following:

    Number of key(s) addedd:        1

    Now try logging into the maschine, with "ssh 'username@server_ipaddress'"
    and check to make sure that only the keys(s) you wanted were added.

After this, the public SSH-Key is successuflly copied to the server. 

## Step 3: Verify the connection by logging into the server with the new SSH-Key
Try to connect to the server specifying the newly created key with following command:

    $ ssh -i ~/.ssh/private_key username@server_ipaddress

If everything was done correctly, a connection with the server is successffuly established. The terminal shows the server "message of the day" and the command line should look like this:

    username@server:~$

> **[!Info]**  
> If a passphrase was given during SSH-Key creation, you will be prompted for it, even through an SSH-Key authentication.

### Step 3.1: Checking the authorized keys on the server
Verify the `authorized_keys` file content on the server using following command:

    $ cat ~/.ssh/authorized_keys

In the output, a list of authorized SSH-Keys is displayed. If the key created and copied earlier has a comment, it'll be quickly recognisable.

## Step 4: Deactivate Password authentication for the SSH Login
For this step, open the SSH configuration file `sshd_config` on the server with following command:

    $ sudo nano /etc/ssh/sshd_config

Search for the line that states `#PasswordAuthentication yes`, remove the `#` and change the `yes` to `no`. Save and close the file and restart the SSH service with:

    $ sudo systemctl restart ssh.service

## Step 5: Configuration Double Check! 
To test the configuration, log out from the server and try to connect again **without** the SSH-Key:

    $ ssh -o PubkeyAuthentication=no username@server_ipaddress

> **[! Info]**  
>`-o` passes a specific configuration option 

The connection should fail showing this message:

    username@server_ipaddress: Permission denied (publickey)

Confirming that the only Authentication method is with a valid SSH key!

# Nginx installation and configuration with a custom index
## Step 1: Install Nginx
In the V-server, first of all, update and upgrade the system, as needed, by running the command `sudo apt update && sudo apt upgrade`.
Afterward, install Nginx:

    sudo apt install nginx -y

> **[! Info]**  
>`-y` for accepting automatically all the prompts during the installation.

Once the installation is complete, check the Nginx status with:

    $ systemctl status nginx.service

The output shows a lot of informations, such as Process, Tasks, Memory, etc.  
Nginx is active if the status is marked as such `Active: active (running)` followed by the date and time.  
If the server IP address is entered in a browser, it should show the welcome page of Nginx, with the message: 

> **Welcome to nginx!**  
> If you see this page, the nginx web server is successfully installed and  
> working. Further configuration is required.  

## Step 2: Create an alternative Index Page
First, create a new folder in the Nginx index directory `/var/www/html/`:

    $ sudo mkdir /var/www/alternatives

Second, create the new `alternate-index` page, with touch:

    $ sudo touch /var/www/alternatives/alternate-index.html

After that open the newly created index, either with nano or another text editor:

    # sudo nano /var/www/alternatives/alternate-index.html

And fill it with html code, for example:

    <!doctype html>
    <html>
    <head>
        <meta charset="utf-8">
        <title>Hello, Nginx!</title>
    </head>
    <body>
        <h1>Hello, Nginx!</h1>
        <p>I have just configured our Nginx web server on Ubuntu Server!</p>
    </body>
    </html>


## Step 3: Create an alternative Nginx configuration file
Add a new `alternatives` config file in the default Nginx configuration directory `/etc/nginx/sites-enabled/`:

    $ sudo nano /etc/nginx/sites-enabled/alternatives

Write following configuration:

    server {
        listen 8081;
        listen [::]:8081;

        root /var/www/alternatives;
        index alternate-index.html;

        location / {
            try_files $uri $uri/ =404;
        }   
    }

In this block:
- the listening port is changed from the default port 80 to port 8081:
    - first line refers to IPv4  
    - second line refers to IPv6

- The `root` line specifies the directory where the important site files are stored.  

- The `index` line specifies the new entry point of nginx, which is the alternative index we created. 

- The last block, `location`, handles user requests to the server. If the requests aren't the root page, nginx will search for a corrisponding element. If the user's requests can't be satisfied, a 404 message is displayed. 

## Step 4: Test the new config
Before restarting Nginx, test if the alternative configuration file is valid or not:

    $ sudo nginx -t

This command checks the configuration files for possible syntax errors.

The output should look like this:

    nginx: the configuration file /etc/nginx/nginx.conf syntax is okay
    nginx: configuration file /etc/nginx/nginx.conf test is successful

> **[!Info]**  
> Even if the alternative configuration file is not showed in the output, it is probably been tested too, since the `nginx.conf` usually contains references to other configurations files, like the ones stored in the directory `/sites-enabled/`.

After passing the test, restart Nginx services:

    $ sudo service nginx restart

Then check again with `systemctl status nginx.service` if the server is running. 

If active and running, open a browser and enter the server IP Address specifying the port 8081 like this: `http://server_ipaddress:8081`.  
The `alternate-index.html` page should be displayed.

Furthermore search for a non existent page, for ex. `http://server_ipaddress:8081/im-not-here`, and a *404 Not Found page* should be displayed.  
If all checks, the alternative configuration works as expected.

# Git configuration on the V-server
## Step 1: Installing git
On the server, first check if Git is installed:

    $ git --version

If Git is already installed, the output will show the version, for ex. `git version 2.24.1`.  
If not, then:

    $ sudo apt update
    $ sudo apt install git

Once installed, set your identity for Git on the v-server through following commands:

    $ git config --global user.name "my github username"
    $ git config --global user.email "my github email account"

Double check if those infos are successfully stored by running:

    $ git config --list

The entered credentials should be displayed.

## Step 2: Create an SSH key pair on the server and copy it on Github
For the creation of the SSH-key pair, follow the same process described in [Step 1](#step-1-create-ssh-key-pair-on-local-host), but do it on the server.

After completion, print the public key's in the terminal:

    $ cat ~/path_to_the_key/key.pub

Copy the key content in the clipboard, open a browser and go to the [Github SSH and GPG keys section of your account](https://github.com/settings/keys).  
Click on `New SSH Key`, add a title, leave the key type to "Authentication key", paste the key and save.

To test the ssh connection to Github, run:

    $ ssh -T git@github.com

If the process was done correctly, the output should look like this:

    Hi my_github_username! You've successfully authenticated, but GitHub does not provide shell access.

Now you are able to use Git directly from the server!

# Workflow improvements

The workflow described in this guide can be of course improved using different strategies. 

## The Alias function
The alias function is a pretty good solution for avoid typing the same long command multiple times.
The syntax is pretty simple:

    $ alias name_of_choice="command line"

For example, to avoid typing the SSH command several times:

    $ alias vserver="ssh -i /path/to/the/private_key username@server_ipaddress"

Now, typing just `vserver`, in the terminal, will establish the SSH connection with the server.

> **[!Info]**  
> The defined alias is temporary. Opening a new terminal session will result in losing that alias. 
> This can be addressed by saving the aliases in the terminal configuration.

## SSH config file and multiple identities
A more specific and long term solution for simplifying the SSH workflow is the use of an SSH config file. In this file you can setup one or multiple identities, each with several configurations for different servers.

First, open the file `config` in the `~/.ssh/` directory, or create one if it doesn't exists:

    $ nano ~/.ssh/config

Write a configuration block, for example:

    Host chosen_server_name
      HostName server_ip_address
      User server_username
      PreferredAuthentications publickey
      IdentityFile ~/path/to/private_key/key

Save and close nano. Now instead of typing the whole ssh command, connect to the server using the alias defined in the `Host` line of the block:

    $ ssh chosen_server_name

SSH will automatically connect using the settings from the configuration block written in `~/.ssh/config`.

The configuration block described can of course be improved with even more options.

## Handling passphrases with the SSH agent
If a passphrase was set when creating an SSH Key pair, it will be asked every time the keys are used.
To avoid this, you can use the programm SSH Agent, which loads your private SSH keys in memory and provide them when asked.

Proceed as follows:

    eval $(ssh-agent -s)
    ssh-add ~/path/to/private/key

The `ssh-add` command will return a prompt asking for the passphrase. After entering it, this is gonna show:

    Identity added: ~/path/to/private/key (Key comment)

Now, the indicated private key is *loaded* in the SSH agent and, when connecting again to the server, no passphrase will be requested.

To check which keys are loaded in the SSH agent, run following command:

    $ ssh-add -l 

> **[!Info]**  
> This process is temporary. Opening a new terminal session will clear the loaded key. You can automate this by saving the line  `$ eval $(ssh-agent -s)` in the terminal configuration, so that the agent starts automatically every time you start a new terminal session (keys still need to be loaded though using `$ ssh-add`).
> This can be addressed entirely by using a keychain or a similar program.
