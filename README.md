DigitalOcean Setup Notes
=========================

> This is just a reminder for when I need to setup a new droplet on
> DigitalOcean. I might make this into a shell script one day.

A lot of this information is based on the great tutorials from the DigitalOcean
community website.

## 1. Droplet Config

**Region**

Use the San Francisco or Singapore regions - fastest download times from NZ.

TODO: post speed test results from all servers

**Image**

Use `Applications > Ubuntu LAMP on 14.04`. 

**SSH Keys**

Choose to use an SSH key instead of a password.

## 2. Ubuntu Config

### Setup User Accounts

[Tutorial](https://www.digitalocean.com/community/tutorials/initial-server-setup-with-ubuntu-14-04)

SSH in as root

    ssh root@hostname

Change root password

    passwd

Create admin user

    adduser admin

Add admin to sudoers file

    EDITOR=vim visudo
    # + admin   ALL=(ALL:ALL) ALL

### Software

Update and upgrade all packages

    sudo apt-get update
    sudo apt-get upgrade
    sudo apt-get autoremove
    sudo apt-get clean

Install useful packages

    sudo apt-get install git make tmux vim zsh ranger gcc silversearcher-ag

Grab dotfiles for tmux and vim

    git clone https://github.com/stayradiated/dotfiles.git ~/dotfiles
    cd dotfiles
    make tmux vim

Setup zsh

    cp ~/dotfiles/crux/zsh/zshrc ~/.zshrc
    vim ~/.zshrc
    chsh -s $(which zsh)

### SSH Setup

Generate a [Random Port
Number](http://www.random.org/integers/?num=1&min=1025&max=65535&col=1&base=10&format=plain&rnd=new).
Remember to store it in your password manager.

Now edit the the sshd config. TODO: make this into a valid patch file.

    # Open SSH config
    vim /etc/ssh/sshd_config

    # Change port
    Port 12345

    # Disable Root Login
    PermitRootLogin No

    # Only allow admin login
    AllowUsers admin

    # Public Key Auth only
    RSAAuthentication yes
    PubkeyAuthentication yes
    PasswordAuthentication no
    UsePAM no

Now login as `admin` (password required at this point)

    ssh admin@hostname

    # create .ssh server
    mkdir -p ~/.ssh

Now open a new terminal and upload your ssh public key

    cat ~/.ssh/id_rsa.pub | ssh admin@hostname 'cat >> .ssh/authorized_keys'

Then test that you can login as admin without using a password

Now restart the ssh server (from root).
And don't close your root terminal.

    service ssh restart

Now test that your ssh settings work

    # should login without password
    ssh -p **** admin@hostname

    # should be blocked
    ssh -p **** root@hostname
    
### Create a Swap Partition

[Tutorial](https://www.digitalocean.com/community/tutorials/how-to-add-swap-on-ubuntu-14-04)

(optional) Check for existing swap partitions

    sudo swapon -s
    free -m

(optional) Check available hard drive space

    df -h

Create swapfile (use the size of your RAM) and check the size

    sudo fallocate -l 1G /swapfile

Set the swapfile permissions

    sudo chmod 600 /swapfile

Check everything is looking allright

    ls -lh /swapfile
    # -rw------- 1 root root 1.0G Nov  1 19:47 /swapfile

Setup the swap space

    sudo mkswap /swapfile
    sudo swapon /swapfile

Check that it is using the swapfile

    sudo swapon -s
    free -m

Make it persist between reboots

    echo "/swapfile   none    swap    sw    0   0" >> /etc/fstab

Make it not use the swapfile unless it really needs it

    sudo sysctl vm.swappiness=10
    echo "vm.swappiness=10" >> /etc/sysctl.conf
    cat /proc/sys/vm/swappiness

Lower cache pressure - keep inode info for longer

    sudo sysctl vm.vfs_cache_pressure=50
    echo "vm.vfs_cache_pressure=50" >> /etc/sysctl.conf
    cat /proc/sys/vm/vfs_cache_pressure

## 3. Silverstripe Setup

[Tutorial](https://www.digitalocean.com/community/tutorials/how-to-install-silverstripe-on-your-vps)

### Apache

Enable `mod_rewrite`

    sudo a2enmod rewrite
    apache2ctl -M | grep rewrite

Editing the Apache default virtual host file.

    sudo vim /etc/apache2/apache2.conf

    <Directory /var/www/>
        Options Indexes FollowSymlinks
        AllowOverride All
        Require all granted
    </Directory>

    sudo vim /etc/apache2/sites-enabled/000-default.conf

    DocumentRoot /var/www/silver

Set PHP timezone

    sudo vim /etc/php5/apache2/php.ini

    date.timezone = Pacific/Auckland

Make sure that silverstripe deps are installed

    sudo apt-get install php5-gd php5-curl php5-tidy

Restart apache

    sudo service apache2 restart

### Silverstripe
    
    cd ~
    mkdir silver
    wget http://www.silverstripe.org/assets/releases/SilverStripe-cms-v3.1.6.tar.gz
    tar -xzvf SilverStripe-cms-v3.1.6.tar.gz -C silver
    sudo mv silver /var/www

    cd /var/www/silver

Configure ownership

    sudo chown -R root:www-data assets
    sudo chown root:www-data .htaccess
    sudo chown root:www-data mysite/_config.php

Configure permissions

    sudo chmod 775 -R assets
    sudo chmod 775 .htaccess
    sudo chmod 775 mysite/_config.php

## MySQL

    # get current root password
    cat /etc/motd.tail | grep password

    mysql -u root -p
        -> show databases;
        -> create database silver;
        -> create user 'webserver'@'localhost' identified by '<password>';
        -> grant all privileges on * . * to 'webserver'@'localhost';
        -> flush privileges;
        -> exit

    mysql_secure_installation
        - enter current mysql root password
        - change root password to something new
        - remove anonymous users
        - disable root login remotely
        - remove test database
        - reload privileges table

## 4. Git Setup

[Tutorial](https://www.digitalocean.com/community/tutorials/how-to-set-up-automatic-deployment-with-git-with-a-vps)

### Server

Create a new git repo

    sudo mkdir -p /var/repo/silver.git
    sudo chown admin:admin /var/repo/silver.git
    cd /var/repo/silver.git
    git init --bare

Setup somewhere to checkout the files

    sudo mkdir -p /var/www/source
    sudo chown admin:admin /var/www/source

Setup the post-receive hook

    touch hooks/post-receive
    chmod +x hooks/post-receive
    vim hooks/post-receive

    --- start ---
    #!/bin/sh
    git --work-tree=/var/www/source --git-dir=/var/repo/silver.git checkout -f

    cd /var/www/source

    if [ -x ./deploy.sh ]; then
        ./deploy.sh /var/www/silver
    fi
    --- end --

### Local

Add the remote to your git repo

    git remote add live ssh://admin@hostname:*****/var/repo/silver.git

Setup a deploy.sh

    touch deploy.sh
    chmod +x deploy.sh
    vim deploy.sh

    --- start ---
    #!/bin/sh

    src=`pwd`
    dst=$1

    echo "Linking $src with $dst/themes/myTheme"
    rm -fr $dst/themes/jpc2014
    ln -s "$src" "$dst/themes/myTheme"

    echo "Linking $src/mysite/code with $dst/mysite/code"
    rm -fr $dst/mysite/code
    ln -s "$src/mysite/code" "$dst/mysite/code"
    --- end --

