# Virtual Exim 2
## README AND INSTALL GUIDE

Thanks for picking the Virtual Exim package, for your virtual mail hosting needs! :-)

This document provides a basic guide on how to get Virtual Exim working on your system. In this guide, I assume that you have *a little* knowledge of both MySQL and Exim.

Before we go into any details, I'd like to thanks Philip Hazel and the Exim developers for a fine product. I would also like to thanks the postmasters at various domains for letting me play havoc with their mail while I set this up :-) Finally, a special note of thanks to Dan Bernstein for his Qmail MTA. Dan, thank you for educating me how mail delivery really shouldn't be done, on the Internet.

The Virtual Exim project currently lives on GitHub: https://github.com/avleen/vexim2
And its mailing list/Google group is available at: https://groups.google.com/group/vexim

### Installation steps for each component:

##### NOTE FOR UPGRADING:
If you are upgrading from a previous version of Virtual Exim, you'll find additional notes marked 'UPGRADING' in some sections. If and when you do, follow these notes.

##### DISTRIBUTION-SPECIFIC NOTES:
Some sections may contain distribution or OS-specific notes. You'll find them after an appropriate prefix, such as 'DEBIAN' or 'FREEBSD' where appropriate.

##### PARTS:
1. Prerequisites
2. Files and Apache
3. System user
4. Databases and authentication
5. Exim configuration
6. Site Admin
7. Virtual Domains
8. Mailman
9. Mail storage and Delivery
10. POP3 and IMAP daemons (separate to this software)

#### Prerequisites:
The following packages must be installed on your system, for Virtual Exim to work. If you don't have any of these packages already installed, please refer to the documentation provided with your operating system on how to install each package:
* Exim v4 with MySQL or PostgreSQL support (tested on v4.1x/4.2x/4.7x)
* MySQL (tested on v5.1.x) or PostgreSQL
* Apache or other HTTP server (Tested on Apache v2.2.x)
* PHP v5 (tested on v5.3.x) with at least the following extensions:
  * PDO
  * pdo_mysql or pdo_pgsql
  * imap
  * gettext
  * iconv

The following packages provide optional functionality:
* Mailman – to have mailing lists
* ClamAV – for scanning e-mail for viruses
* SpamAssassin – for scanning e-mail from spam

VExim might work with older (or newer) versions of these packages, but you may have to perform some adaptation work to achieve that. In any case, you are welcome to file bugs and/or provide patches on GitHub.

**UPGRADING:** If you are upgrading from VExim 1.x, the following packages will help you to perform automatic database schema migration. They will only be needed during the upgrade and can be removed later:
* Perl + DBI module + DBD-mysql (you would need DBD-Pg to migrate a PostgreSQL database, but PostreSQL migration is not implemented in the migration script)

**DEBIAN:** The following command line installs all the required packages (this is assuming you're going with MySQL setup):
```
# apt-get install apache2 exim4-daemon-heavy mysql-server libapache2-mod-php5 php5-mysql php5-imap
```
If you want your mail server to scan messages for viruses and spam, you will have to install a few more packages:
```
# apt-get install clamav-daemon clamav-freshclam sa-exim spamassassin
```

##### System user:
You should create a new user account to whom the virtual mailboxes will belong. Since you do not want anyone to be able to login using that account, you should also disable logging in for that user. Here are the command lines to do that. This manual assumes you want to have your virtual mailboxes in /var/vmail. If you want them elsewhere, adjust the commands. After the user and group are created, find their uid and gid using the last command and memorize these values:
```
# useradd -r -m -U -s /bin/false -d /var/vmail vexim
# id vexim
```
**FREEBSD:** Instead of the commands above, you should probably use the following:
```
# pw useradd vexim -u 1,100 -g "" -d /var/vmail -m -s /nonexistant
# id vexim
```
**DEBIAN:** Use the following command instead:
```
# adduser --system --home /var/vmail --disabled-password --disabled-login --group vexim
```

##### Databases and authentication:
When creating the databases you have two options. You can either use the SQL command files, or the perl script. If you are creating new databases, I *HIGHLY* recommend you use the SQL command files. They are much simpler. However, if you are migrating from Virtual Exim 1.x to 2.x, you will need to use the perl script to migrate the data.

**IMPORTANT NOTE:** if you have a database called "vexim" in MySQL or PostgreSQL, you should back it up first. The SQL command files assume that the database with that name does not exist and has to be created. You can edit the files to use a different database name, but you are still strongly advised to make a backup of the "vexim" database – just in case.

##### MySQL:
This ditribution contains a file "vexim2/setup/mysql.sql". This file provides the database schema used by vexim. You will have to import it into MySQL, but before that, some changes must be made to this file in order for it to work.

Open this file in a text editor and look for the word "CHANGE". You will find the following two lines not far from the top (lines 15-16):
```mysql
uid              smallint(5)   unsigned  NOT NULL  default 'CHANGE',
gid              smallint(5)   unsigned  NOT NULL  default 'CHANGE',
```
Replace "CHANGE" with the uid and gid of your freshly created "vexim" user (see the System user step). This will be the default user to own mailbox files for new domains.

Further below you will find the following section:
```mysql
--
-- Privileges:
--
GRANT SELECT,INSERT,DELETE,UPDATE ON `vexim`.* to "vexim"@"localhost"
    IDENTIFIED BY 'CHANGE';
FLUSH PRIVILEGES;
```
Replace the word "CHANGE" with a secure password - It will be used both by the Exim MTA and the web interface to access the database.

If the crypt() function on your system produces DES hashes, you will also have to uncomment the appropriate lines, as noted by the comments, at the end of mysql.sql.

With the necessary changes made, you should run the following command line to initialize the database:
```
# mysql -u root -p < vexim2/setup/mysql.sql
```

##### PGSQL:
The code has been tested by several users to work with Virtual Exim, and we try our best to make sure it always will. Unfortunately I don't have much PostgreSQL knowledge to support it fully. A database schema for it is included however, as setup/pgsql.sql to help you set up the database. Make sure to adjust it similarly as per MySQL instructions above.

**UPGRADING:**
If you are upgrading your installation you will need to use the perl script. Executing it as:
```
# create_db.pl --act=migratemysql --dbtype=mysql --uid=90 --gid=90 --mailstore=/var/vmail
```
should work fine. Replace 'uid', 'gid' and 'mailstore' values with the ones you have from the "System user" step.


#### Files and Apache:
In this distribution is a directory called 'vexim'. You have two options:
* Copy this directory into your current DocumentRoot for your domain, and optionally rename the directory.
* Set up a new VirtualHost and point the DocumentRoot to the vexim directory.

Both should work equally well.

After copying the 'vexim' directory, you should find the 'variables.php.example', file in its subdirectory called 'config', copy that file to 'variables.php' and change the following values defined in it:
* $sqlpass – to the vexim database user's password which you chose while editing 'mysql.sql' in the "Databases and authentication" step.
* $uid, $gid and $mailroot to the values you have from the "System user" step.
* $cryptscheme is set to "sha512", a more specific configuration or other crypt-schemes can be used.

Other, less interesting options are documented in the comments of that file. Feel free to explore them as well.


##### Exim configuration:
**NOTE:** the configuration files supplied here have been revised. You should use them carefully and report problems!

An example Exim 'configure' file, has been included with this distribution as 'docs/configure'. Copy this to the location Exim expects its configuration file to be on your installation. You will also need to copy docs/vexim* to /usr/local/etc/exim/. The following lines are important and will have to be edited if you are using this configure, or copied to your own configure file:


Edit these if your mailman is in a different location:
```
MAILMAN_HOME=/usr/local/mailman
MAILMAN_WRAP=MAILMAN_HOME/mail/mailman
```
These need to match the username and group under which exim runs:
```
MAILMAN_USER=mailnull
MAILMAN_GROUP=mail
```
Change this to the name of your server:
```
primary_hostname=mail.example.org
```
If you are using MySQL, uncomment the following two lines:
```
#VIRTUAL_DOMAINS = SELECT DISTINCT CONCAT(domain, ' : ') FROM domains type = 'local'
#RELAY_DOMAINS = SELECT DISTINCT CONCAT(domain, ' : ') FROM domains type = 'relay'
```
If you are using PGSQL, uncomment the following four lines:
```
#VIRTUAL_DOMAINS = SELECT DISTINCT domain || ' : ' FROM domains WHERE type = 'local'
#RELAY_DOMAINS = SELECT DISTINCT domain || ' : ' FROM domains WHERE type = 'relay'
```
Depending on the database type you are using, you will need to uncomment the appropriate lines in the config, to enable lookups.

These control which domains you accept mail for and deliver locally (local_domains), which domains you accept mail for and deliver remotely (relay_to_domains), which IP addresses are allowed to send mail to any domain (relay_from_hosts) and which system users are considered trusted (trusted_users). More on these options – in Exim documentation. 
```
domainlist local_domains = @ : example.org : ${lookup mysql{VIRTUAL_DOMAINS}} : ${lookup mysql{ALIAS_DOMAINS}}
domainlist relay_to_domains = ${lookup mysql{RELAY_DOMAINS}}
hostlist   relay_from_hosts = localhost : @ : 192.168.0.0/24
#trusted_users = www-data
```
Specify here, the username and group under which Exim runs. This combination is also that under which mailman must run in order to work:
```
exim_user = mailnull
exim_group = mail
```
If you want to use either Anti-Virus scanning, or SpamAssassin, you will need to uncomment the appropriate line here.
```
# av_scanner = clamd:/tmp/clamd
# spamd_address = 127.0.0.1 783
``` 
This line configures database connectivity. You need to uncomment it and change the word 'CHANGE', to the password you will use for the 'vexim' database user, which we will set up in the next part.
```
#hide mysql_servers = localhost::(/tmp/mysql.sock)/vexim/vexim/CHANGE
```
Also it is assumed that the mysql domain socket is /tmp/mysql.sock, which is where the FreeBSD port puts it. Other installations put it in /var/tmp, /usr/lib, or any number of other places. If yours isn't /tmp/mysql.sock, you will need to set this.

TLS is activated by default. We suppose that you already created a SSL key and certificate. 
```
tls_certificate = /etc/exim4/exim.crt
tls_privatekey = /etc/exim4/exim.key
```
The creation of SSL-keys is the same like for webservers, e.g. https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-with-a-free-signed-ssl-certificate-on-a-vps . You can use the same certificate your the webserver of this host (if you use webmail).

```
tls_dhparam = /etc/exim4/dhparam.pem
```
The Diffie-Hellman group should have at least 1024 bit and can be created with this command (it can take some time):
```
# openssl dhparam -out /etc/exim4/dhparam.pem 2048
```

###### ACL's: 
We have split all of the ACL's into separate files, to make managing them easier. Please review the ACL section of the configure file. If there are ACL's you would rather not have executed, please comment out the '.include' line that references them, or edit the ACL file directly and comment them out.

###### DEBIAN:
Typically, Debian setups use split Exim configuration with some Debconf magic. This manual will assume that you are familiar with it. If not, you should refer to the Debian documentation on Exim. To get the virtual mailboxes to work, copy the contents of docs/debian-conf.d/ to /etc/exim4/conf.d/ and change the MySQL password in .../main/00_vexim_listmacrosdefs. You may also want to review the ACL's in docs/vexim-acl-*.conf and selectively copy and paste their contents to the files provided by Debian in conf.d. By the way, some of these ACL's are already implemented by Debian, so you might just need to enable them by defining certain macros as described in Debian manual. This manual does not cover enabling ClamAV and SpamAssassin in Exin in Debian. Please look this up elsewhere. By the way, the author of this part never bothered to set up Vexim in such a way that Debian would take into account the status of the various user flag (on_av, on_spamassassin etc) for each user. In his setup, these flags have no effect, and all messages are checked for spam and viruses.

Stefan Tomanek has a nice writeup about using Vexim in Debian, but that article does not cover all aspects, is a bit outdated, and most of if has been incorporated (and improved!) into this document anyway. You can find it at http://stefans.datenbruch.de/rootserver/vexim.shtml.


#### Site Admin:
In order to add and delete domains from the database, you need to have a "site admin". This user can create the initial postmaster users for the individual domains.

The default username and password for the siteadmin, are:
* **Username:** siteadmin
* **Domain value:** (blank)
* **Password:** CHANGE

The password is case sensitive. You are strongly advised to log in and change it as soon as you get a chance. :-)

#### Virtual Domains:
Virtual Exim can now control which local domains Exim accepts mail for and which domains it relays mail for. The features are controlled by the siteadmin, and domains can be easily added/removed from the siteadmin pages. Local domains can also be enabled/disabled on the fly, but relay domains are always enabled.

#### Mailman:
Mailman needs to be installed if you want to use mailing lists. The default location is assumed to be /usr/local/mailman. If this is not the location of your installation, edit Exim's configure file, and change the paths where ever 'mailman' is mentioned, and do the same in vexim/config/variables.php

#### Mail storage and Delivery:
The mysql configuration assumes that mail will be stored in /var/vmail/domain.com/username/Maildir. If you want to change the path from '/var/vmail/', you need to edit the file:
>vexim/config/variables.php

and change 'mailroot' to the correct path. Don't forget the / at the end.

#### POP3 and IMAP daemons:
There are many POP3 and IMAP daemons available today. Few of them are good, and fewer of those like MySQL. Some that we have found that work are:
* **POP3 Only:** Qpopper
* **IMAP and/or POP3:**  The Courier-IMAP package or Dovecot

Instructions for installing these have been included in this tarball in the following files:
* **Qpopper:** docs/clients/qpop-mysql.txt
* **Courier-IMAP:** docs/clients/courierimap.txt
* **Dovecot:** docs/clients/dovecot.txt

These documents are pretty clear and you should be able to use them as a template when compiling from source on most Unixes. Just remember the switches on 'configure' scripts for enabling mysql support :-)

Instructions for configuring Cyrus and Cyrus IMAP are also available. However, we have not tested these so cannot guarantee they work. If you have success or problems with these instructions, please let us know!
* **Cyrus:** docs/clients/cyrus.txt

**UPGRADING:** If you are upgrading, you will need to update your configs for your POP/IMAP daemons, as the database layout has changed. You should be able to follow the above instructions without problem.
