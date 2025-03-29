# v-server-setup
Project nr. 1. during my time at Developer Akademie

# Table of Contents

1. [Introduction](#introduction)  
2. [Starting Requirements](#starting-requirements)  
3. [Main Objective of this Project](#main-objective-of-this-project)  

## Steps:

1. [Step 1: Testing the given login credentials](#step-1-testing-the-given-login-credentials)
2. [Step 2: Creating an SSH Key pair on main machine](#step-2-creating-a-ssh-key-pair-on-main-maschine)
3. [Step 3: Saving the SSH public key on the server (for SSH Key login authentication)](#step-3-saving-the-ssh-public-key-on-the-server-for-ssh-key-login-authentication)
4. [Step 4: Deactivation of server User-Login Authentication](#step-4-deactivation-of-server-user-login-authentication)
   - [Time for a Double Check!](#time-for-a-double-check)
5. [Step 5: Nginx Installation](#step-5-nginx-installation)
6. [Step 6: Creating an alternative configuration and Index Page for Nginx](#step-6-creating-an-alternative-configuration-and-an-alternate-index-page-for-nginx)
   - [Step 6.1: Creating the alternative nginx config file](#step-61-creating-the-alternative-nginx-config-file)
   - [Step 6.2: Writing the alternate-index.html](#step-62-writing-the-alternate-indexhtml)
   - [Step 6.3: Testing the new config and restarting Nginx services](#step-63-testing-the-new-config-and-restarting-nginx-services)
7. [Step 7: Setting git/github access from the v-server through SSH keys](#step-7-setting-gitgithub-access-from-the-v-server-through-ssh-keys)
    - [Step 7.1: Creating an SSH key pair on the server and saving it on Github](#step-71-creating-an-ssh-key-pair-on-the-server-and-saving-it-on-github)

## Extra:

1. [Quality of life commands](#quality-of-life-commands)
2. [Setting up the ssh config file](#setting-up-the-ssh-config-file)
    - [Setting up the ssh agent](#setting-up-the-ssh-agent)
3. [Using the Alias function](#using-the-alias-function)


## Introduction
In this document I am going to document all the steps that i took in order to fulfill my first project at the Developer Akademie, the setup of a virtual server.

## Starting requirements
In order to start this project i was given the IP Address of a V-Server and the login credentials (username and password) to connect to it.

## Main Objective of this Project
Following the checklist created for the project, the main objectives are:

    1. Configuration of the V-server so that the only authentication method is through ssh keys.
    2. Nginx installations and subsequent webserver setup
    3. Setting an alternative configuration and an alternate Index to the Nginx default
    4. Setting git access from the server through ssh keys


## Step 1: Testing the given login credentials
First of all I tried to establish a secure connection to the V server using the provided credentials. In order to do this I used following command in the VS Code terminal:

    ssh username@server_ipaddress 

The command was successful, but since it was my first time connecting to this server, the output informed me that the server was not a known host and prompted me to accept(or not) the new fingerprint.
After answering yes, I entered my password and I was successfully connected to the server.

## Step 2: Creating a SSH Key pair on main maschine
For future ssh key login instead of password, I created a ssh key pair on my computer using following terminal command:

    ssh-keygen -t ed25519 -C "DevAkad_Project"

`-t` specifies the encryption cipher  
`-C` adds a comment at the end of the key (useful to keep track of the key purpose, for ex. for this project)

The terminal then asked me where and in which file i wanted to save the new key pair. I left it blank to let it save it to the default location `~/.ssh/`
Afterwards I was prompted to enter a passphrase, for an extra security layer, which I did (but it can be left blank). After confirming those prompts the key pair was successfully created.
In the terminal the new key fingerprint was displayed.

To double check that I did everything correctly, I listed the ssh key folder's content with the command:

    ls ~/.ssh/

This command showed both my private (`key`) and my public key (`key.pub`), confirming that the SSH key pair was successfully created.

## Step 3: Saving the SSH public key on the server (for SSH Key login authentication)
Following along the study material, it was time to copy the public key onto the server. To do this I used the `ssh-copy-id` program and fired this command in the terminal:

    ssh-copy-id -i ~/.ssh/key.pub username@server_ipaddress

The terminal showed a log from `ssh-copy-id` and then displayed a message:

    Number of key(s) addedd:        1

    Now try logging into the maschine, with "ssh 'username@server_ipaddress'"
    and check to make sure that only the keys(s) you wanted were added.

The key seemed successuflly copied. To double check I logged into the server using the new SSH key with this command:

    ssh -i ~/.ssh/private_key username@server_ipaddress

`-i` specifies the identity 

Since I setted a passphrase for my SSH keys, i was asked for it. After entering the passphrase, I was then successufully connected to the server without needing to enter the server credentials (If I had let the key passphrase blank, the connection would have been right away)

To ensure that only my intended SSH key was copied, I listed the content of the server's home folder and searched for the `.ssh` directory with this command:

    ls -al ~/

`-al` are two short modifiers for `--all`, for including hidden files, and `--long` for returning a detailed list.

After confirming the presence of the `.ssh` folder, I listed its contents and found two directories and a file named `authorized_keys`  

Using the `cat` program I then printed this file's content in the terminal:

    cat ~/.ssh/authorized_keys

In the output, I could see exactly the SSH key I had copied earlier, easily recognizable by the comment I appended at the end of the key (or by comparing it to the key.pub file on my machine).

## Step 4: Deactivation of server User-Login Authentication

While being secure connected to the server, I searched for the `sshd_config` file, which handles SSH configuration on the server.
From the study material, I learned that this file is located at `/etc/ssh/`, so i ran following command in the terminal:

    sudo nano /etc/ssh/sshd_config

Once the file was open, I searched for the line that states `#PasswordAuthentication yes`. I then removed the `#` and changed the `yes` in `no`. Afterward, I saved and closed the file and restarted the SSH service with:

    sudo systemctl restart ssh.service

### Time for a Double Check! 
I logged off from the server and tried to connect again, excluding the SSH key authentication and therefore forcing for another authentication method:

    ssh -o PubkeyAuthentication=no username@server_ipaddress
`-o` passes a specific configuration option. In this case telling SSH to not use public key authentication. 

The output for this was:

    username@server_ipaddress: Permission denied (publickey)

Confirming that only Authentication method is with a valid SSH key!

## Step 5: Nginx Installation

In the V-server, before proceeding to install Nginx I checked for updates and upgraded packages as needed by running the command `sudo apt update && sudo apt upgrade`.
After the update and upgrade I then ran:

    sudo apt install nginx -y

`-y` tells `apt` to accept automatically all the prompts during the installation.

Once the installation was complete, I checked if nginx was running with:

    systemctl status nginx.service

The output showed a lot of informations, such as Process, Tasks, Memory, etc., witt the status highlighted as:
`Active: active (running)` followed by the date and time, confirming that nginx was successful installed and running.  

Entering then the IP address of the server in a browser, showed me the welcome page of nginx with the message: 

> **Welcome to nginx!**  
> If you see this page, the nginx web server is successfully installed and  
> working. Further configuration is required.  

## Step 6: Creating an alternative configuration and an alternate Index Page for Nginx

To set an alternative index, I first created an alternative folder in the Nginx index directory `/var/www/html/` with following command:

    sudo mkdir /var/www/alternatives

After that, to create the new index alternative page, I used the touch command with the following line:

    sudo touch /var/www/alternatives/alternate-index.html

### Step 6.1: Creating the alternative nginx config file
After creating these file/folders, the next step was to create and add an alternative nginx configuration. I did this by adding a new `alternatives` config file in the default nginx configuration directory `/etc/nginx/sites-enabled/` through nano:

    sudo nano /etc/nginx/sites-enabled/alternatives

Nano opened and I wrote following configuration (taken from the study material) in the file:

    server {
        listen 8081;
        listen [::]:8081;

        root /var/www/alternatives;
        index alternate-index.html;

        location / {
            try_files $uri $uri/ =404;
        }   
    }

With the first block the listening port is changed from the default port 80 to port 8081:
- first line refers to IPv4  
- second line refers to IPv6

The `root` line specifies the directory where the important site files are stored. 

The `index` line specifies the new entry point of nginx, which is the alternative index we created. 

The last block, `location`, handles user requests to the server. If the requests aren't the root page, nginx will search for a corrisponding element. If the user's requests can"t be satisfied, a 404 message is gonna be displayed. 

### Step 6.2: Writing the alternate-index.html
Time to modify the `alternate-index.html` file created earlier:

    sudo nano /var/www/alternatives/alternate-index.html

From the study material I passed following code:

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

I then saved and closed nano. Everything was set.

### Step 6.3: Testing the new config and restarting Nginx services
Before restarting Nginx I tested if the alternative configuration file was valid or not through following terminal command:

    sudo nginx -t
`-t` tests the configuration files for possible syntax errors before even restarting the service

The output looked promising:

    nginx: the configuration file /etc/nginx/nginx.conf syntax is okay
    nginx: configuration file /etc/nginx/nginx.conf test is successful

However I couldnt quite understand how my configuration file was tested, since the output displayed only the main `/nginx.conf`. 
After some  web searching, I then learned that `nginx.conf` usually contains references to other configurations files, like the ones stored in the directory `/sites-enabled/`.
This meaning that even though the output only mentioned the main config file, the test included the alternatives too!

Since the test was passed I restarted Nginx services:

    sudo service nginx restart

and checked again with `systemctl status nginx.service` displaying an `active (running)` status!

Afterward, I opened a browser and entered the server IP Address specifying the port 8081 like this: `http://server_ipaddress:8081`.  

This displayed the `alternate-index.html` as expected.
Furthermore I added to that a non existent page to the url like `http://server_ipaddress:8081/im-not-here` and a 404 Not Found page was successfully displayed.

## Step 7: Setting git/github access from the v-server through SSH keys

First I checked if git was already installed:

    git --version

The output confirmed it by showing `git version 2.34.1`.  
Next, I set the my identity for git on the v-server through following commands:

    git config --global user.name "my github username"
    git config --global user.email "my github email account"

I then checked if those infos were saved by running:

    git config --list

My credential were correctly und successfully displayed.

### Step 7.1: Creating an SSH key pair on the server and saving it on Github

For the creation of the key pair, I followed the same process described in [Step 2](#step-2-creating-a-ssh-key-pair-on-main-maschine) and simply changed the comment at the end of the key to something more pertinent to its function:

    ssh-keygen -t ed25519 -C "Github_FromServer"

After successfully creating the keys, I printed the public key's content in the terminal using the `cat` command:

    cat ~/path_to_the_key/key.pub

I then copied the key and navigated to Github.com under SSH and GPG keys.
I added a title, leaved the key type  to "Authentication key", pasted the key and saved.

For a double check I then tested the ssh connection to Github through this command:

    ssh -T git@github.com

And obtained following output confirming that the process was done correctly:

    Hi my_username! You've successfully authenticated, but GitHub does not provide shell access.

## Quality of life commands

While writing this document and testing everything described, I found myelf entering the ssh login command several times. Since it's not a short one, I started looking for a way to automate this process. I then found at the end of the module's study material exactly what I was looking for: the configuration of multiple identities in the ssh config file and the `alias` terminal function. After reviewing these, I did logically set up first the configuration file and then created an alias.

## Setting up the ssh config file

First, I searched for the config file in the `~/.ssh/` and, since there was none, I created one:

    nano ~/.ssh/config

In it I wrote a configuration block like this:

    Host server_ip_address
      User server_username
      PreferredAuthentications publickey
      IdentityFile ~/path/to/private_key/key

I saved and closed nano and then ran this command to test if the configuration file was working:

    ssh server_ip_address

I got asked for the key passphrase, entered it and I was successfully connected to the server! 
However entering the key passphrase every time I connected to the server, started getting slightly annoying, so I searched for a solution.

### Setting up the ssh agent

To solve this, I found that there's the need of two commands involving `eval` and the programm `ssh-add`.
So I proceeded as following:

    eval $(ssh-agent -s)
    ssh-add ~/path/to/private/key

After running the `ssh-add ~/path/to/private/key` I was prompted for the passphrase and after entering it, I received this message:

    Identity added: ~/path/to/private/key (Key comment)

For double checking if I did everything right, I also ran following command to check the list of saved identities:

    ssh-add -l 

The corrisponding public key to the private key I added was shown.

For what I understood, the first line set the necessary variables in the shell that let the `ssh-agent` work properly. 
The second line let the `ssh-add` programm connect with the `ssh-agent` and load the key we indicated.

Afterwards, I tried connecting to the server again and there was no need to input my ssh key passphrase, it connected right away!

## Using the Alias function

I found the alias function a pretty good solution in this case, in order to avoid typing long commands multiple times. So I created an alias to shorten the ssh connection process.
The syntax was pretty simple, following along the study material I just wrote following command:

    alias vserver="ssh -i /path/to/the/private_key username@server_ipaddress"

Since I had set up earlier the ssh config file, to connect to server using just the command `ssh server_ip_address`, I updated the alias like this:

    alias vserver="ssh server_ip_address"

I ran `vserver` in the terminal and got connected to the server without the need to enter either the server ip address or the key passphrase.
