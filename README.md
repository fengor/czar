
Introduction
============

Czar is a set of [Ansible](http://ansible.com) playbooks that you can use to build and maintain your own [personal cloud](http://www.urbandictionary.com/define.php?term=clown%20computing) based entirely on open source software, so you’re in control.

It is heavily based on [Sovereign](https://github.com/sovereign/sovereign) from which most of the
code is adapted. However several different software choices have been made which are the main reason
for a new project instead of a personal fork.

If you’ve never used Ansible before, you might find these playbooks useful to learn from, since they show off a fair bit of what the tool can do.

The original author's [background and motivations](https://github.com/sovereign/sovereign/wiki/Background-and-Motivations) might be of interest. tl;dr: frustrations with Google Apps and concerns about privacy and long-term support.

Czar offers useful cloud services while being reasonably secure and low-maintenance. Use it to set up your server, SSH in every couple weeks, but mostly forget about it.

Services Provided
-----------------

What do you get if you point Czar at a server? All kinds of good stuff!

## Mail
-   [IMAP](https://en.wikipedia.org/wiki/Internet_Message_Access_Protocol) over SSL via [Dovecot](http://dovecot.org/), complete with full text search provided by [Solr](https://lucene.apache.org/solr/).
-   [POP3](https://en.wikipedia.org/wiki/Post_Office_Protocol) over SSL, also via Dovecot
-   [SMTP](https://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) over SSL via Postfix, including a nice set of [DNSBLs](https://en.wikipedia.org/wiki/DNSBL) to discard spam before it ever hits your filters.
-   Virtual domains for your email, backed by [PostgreSQL](http://www.postgresql.org/).
-   Spam fighting via [Rspamd](https://www.rspamd.com/).
-   Mail server verification using [DKIM](http://www.dkim.org/) and [DMARC](http://www.dmarc.org/) so the Internet knows your mailserver is legit.
-   Secure on-disk storage for email and more via [EncFS](http://www.arg0.net/encfs).
-   Webmail via [Rainloop](http://www.rainloop.net/).
-   Email client [automatic configuration](https://developer.mozilla.org/en-US/docs/Mozilla/Thunderbird/Autoconfiguration).

## Web
-   Web hosting (ex: for your blog) via [Caddy](https://caddyserver.com/).
-   [CalDAV](https://en.wikipedia.org/wiki/CalDAV) and [CardDAV](https://en.wikipedia.org/wiki/CardDAV) to keep your calendars and contacts in sync, via [nextCloud](http://nextcloud.com/).
-   Your own private storage cloud via [nextCloud](http://nextcloud.org/).
-   Git hosting via [gitea](https://gitea.io/)

## General
-   [Monit](http://mmonit.com/monit/) to keep everything running smoothly (and alert you when it’s not).
-   [collectd](http://collectd.org/) to collect system statistics.
-   Firewall management via [Uncomplicated Firewall (ufw)](https://wiki.ubuntu.com/UncomplicatedFirewall).
-   Intrusion prevention via [fail2ban](http://www.fail2ban.org/) and rootkit detection via [rkhunter](http://rkhunter.sourceforge.net).
-   SSH configuration preventing root login and insecure password authentication
-   [RFC6238](http://tools.ietf.org/html/rfc6238) two-factor authentication compatible with [Google Authenticator](http://en.wikipedia.org/wiki/Google_Authenticator) and various hardware tokens
-   Nightly backups to [Tarsnap](https://www.tarsnap.com/).
-   A bunch of nice-to-have tools like [mosh](http://mosh.mit.edu) and [htop](http://htop.sourceforge.net) that make life with a server a little easier.
-   Your own VPN server via [Wireguard](http://wireguard.com/)

Don’t want one or more of the above services? Comment out the relevant role in `site.yml`. Or get more granular and comment out the associated `include:` directive in one of the playbooks.

Usage
=====

What You’ll Need
----------------

1.  A VPS (or bare-metal server if you wanna ball hard). My VPS is hosted at [Netcup](http://www.netcup.de/). You’ll probably want at least 512 MB of RAM between Apache, Solr, and PostgreSQL. Mine has 1024.
2.  [64-bit Debian 10](http://www.debian.org/) or an equivalent Linux distribution. (You can use whatever distro you want, but deviating from Debian will require more tweaks to the playbooks. See Ansible’s different [packaging](http://docs.ansible.com/ansible/list_of_packaging_modules.html) modules.)
3.  A [Tarsnap](http://www.tarsnap.com) account with some credit in it. You could comment this out if you want to use a different backup service. Consider paying your hosting provider for backups or using an additional backup service for redundancy.

You do not need to acquire an SSL certificate.  The SSL certificates you need will be obtained from [Let's Encrypt](https://letsencrypt.org/) automatically when you deploy your server.


Installation
------------

## On the remote server

The following steps are done on the remote server by `ssh`ing into it and running these commands.

### 1. Install required packages e.g `aptitude` is required on Debian

    apt-get install sudo python

### 2. Get a Tarsnap machine key

If you haven’t already, [download and install Tarsnap](https://www.tarsnap.com/download.html), or use `brew install tarsnap` if you use [Homebrew](http://brew.sh).

Create a new machine key for your server:

    tarsnap-keygen --keyfile roles/tarsnap/files/decrypted_tarsnap.key --user me@example.com --machine example.com

Download a copy of this key and keep it somewhere safe!  There's no point having backups if you can't retrieve them when needed.

### 3. Prep the server

For goodness sake, change the root password:

    passwd

Create a user account for Ansible to do its thing through:

    useradd deploy
    passwd deploy
    mkdir /home/deploy

Authorize your ssh key if you want passwordless ssh login (optional):

    mkdir /home/deploy/.ssh
    chmod 700 /home/deploy/.ssh
    nano /home/deploy/.ssh/authorized_keys
    chmod 400 /home/deploy/.ssh/authorized_keys
    chown deploy:deploy /home/deploy -R
    echo 'deploy ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/deploy

Your new account will be automatically set up for passwordless `sudo`. Or you can just add your `deploy` user to the sudo group.

    adduser deploy sudo

## On your local machine

Ansible (the tool setting up your server) runs locally on your computer and sends commands to the remote server. Download this repository somewhere on your machine, either through `Clone or Download > Download ZIP` above, `wget`, or `git` as below
    
    git clone https://github.com/fengor/czar.git

### 4. Configure your installation

Modify the settings in the `group_vars/czar` folder to your liking. If you want to see how they’re used in context, just search for the corresponding string.
All of the variables in `group_vars/czar` must be set for czar to function.


Finally, replace the `host.example.net` in the file `hosts`. If your SSH daemon listens on a non-standard port, add a colon and the port number after the IP address. In that case you also need to add your custom port to the task `Set firewall rules for web traffic and SSH` in the file `roles/common/tasks/ufw.yml`.

### 5. Set up DNS

Create `A` or `CNAME` records which point to your server's IP address:

* `example.com`
* `mail.example.com`
* `blog.example.com` (for Web hosting)
* `autoconfig.example.com` (for email client automatic configuration)
* `cloud.example.com` (for nextCloud)
* `git.example.com` (for gitea)

### 6. Run the Ansible Playbooks

First, make sure you’ve [got Ansible 1.9.3+ installed](http://docs.ansible.com/intro_installation.html#getting-ansible).

To run the whole dang thing:

    ansible-playbook -i ./hosts --ask-sudo-pass site.yml
    
If you chose to make a passwordless sudo deploy user, you can omit the `--ask-sudo-pass` argument.

To run just one or more piece, use tags. I try to tag all my includes for easy isolated development. For example, to focus in on your firewall setup:

    ansible-playbook -i ./hosts --tags=ufw site.yml

You might find that it fails at one point or another. This is probably because something needs to be done manually, usually because there’s no good way of automating it. Fortunately, all the tasks are clearly named so you should be able to find out where it stopped. I’ve tried to add comments where manual intervention is necessary.

The `dependencies` tag just installs dependencies, performing no other operations. The tasks associated with the `dependencies` tag do not rely on the user-provided settings that live in `group_vars/czar`. Running the playbook with the `dependencies` tag is particularly convenient for working with Docker images.

### 7. Finish DNS set-up

Create an `MX` record for `example.com` which assigns `mail.example.com` as the domain’s mail server.

To ensure your emails pass DKIM checks you need to add a `txt` record. The name field will be `default._domainkey.EXAMPLE.COM.` The value field contains the public key used by DKIM. The exact value needed can be found in the file `/var/lib/rspamd/dkim/EXAMPLE.COM.default.txt`. It will look something like this:

    v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDKKAQfMwKVx+oJripQI+Ag4uTwYnsXKjgBGtl7Tk6UMTUwhMqnitqbR/ZQEZjcNolTkNDtyKZY2Z6LqvM4KsrITpiMbkV1eX6GKczT8Lws5KXn+6BHCKULGdireTAUr3Id7mtjLrbi/E3248Pq0Zs39hkDxsDcve12WccjafJVwIDAQAB

For DMARC you'll also need to add a `txt` record. The name field should be `_dmarc.EXAMPLE.COM` and the value should be `v=DMARC1; p=none`. More info on DMARC can be found [here](https://dmarc.org).

Set up SPF and reverse DNS [as per this post](http://sealedabstract.com/code/nsa-proof-your-e-mail-in-2-hours/). Make sure to validate that it’s all working, for example, by sending an email to <a href="mailto:check-auth@verifier.port25.com">check-auth@verifier.port25.com</a> and reviewing the report that will be emailed back to you.

### 8. Miscellaneous Configuration

To access the server monitoring page, use a SSH tunnel:

    ssh deploy@example.com -L 2812:localhost:2812

Proceeding to http://localhost:2812 in your web browser.


### Reboots

You will need to manually enter the password for any encrypted volumes on reboot. This is not Czar-specific, but rather a function of how EncFS works. This will necessitate SSHing into your machine after reboot, or accessing it via a console interface if one is available to you. Once you're in, run this:

    encfs /encrypted /decrypted --public

It is possible that some daemons may need to be restarted after you enter your password for the encrypted volume(s). Some services may stall out while looking for resources that will only be available once the `/decrypted` volume is available and visible to daemon user accounts.

