---
title: SMTP Relaying for Phishing
author: Babligan
date: 2019-08-11 00:34:00 +0800
categories: [Infra]
tags: [smtp, phishing, postfix]
---

For anyone who's tried to build phishing infrastucture that's consistently reliable, you know by now how challenging that can be. First of all, a proper and functioning mail server has many moving parts that have to be perfect for mail to be sent correctly. At the moment, there are many ways to setup a fully functional mailserver but I generally prefer [iRedMail](https://www.iredmail.org/). The installation is mostly straight forward ([this](https://www.linuxbabe.com/mail-server/ubuntu-18-04-iredmail-email-server) is a great guide if you aren't too familiar with the process). Additionally, iRedmail integrates Nginx, Dovecot IMAP server, Postfix SMTP server, OpenLDAP, MySQL/MariaDB or PostgreSQL for storing user information and a bunch of other incredible features.

Usually, a mail server requires the following to be 'functional':
- Mailing domain name in good reputation (correct categorization, not blacklisted and age).
- Other than the usual A (or AAAA, if you are using IPv6) and MX records, your DNS records have to be:
	- SPF compliant (if your server uses both IPv4 and IPv6, remember to have an inclusive record otherwise you'll get a softfail error i.e `v=spf1 mx ip4:IPv4-ADDRESS ip6:IPv6-ADDRESS ~all`)
	- DKIM signed
	- Conforms to DMARC
	- Domain has a PTR record which resolves as expected.
- SSL/TLS certificates that have to be configured for Nginx/Apache, Dovecot and Postfix.
- Remove postfix email headers.
- While this isn't mandatory, configure port 587 as your primary SMTP rather than port 25. 
	
Robust phishing infrastructure usually incooporates a phishing originator either running Gophish or PhishingFrenzy and a separate mail server and implementing the use of a SMTP relay that sits in front of the phishing originator. However, some organizations have hardened spam filters that can be incredibly difficult to get around. Even after implementing all the requirements, while they can help, they ultimately may not be enough. 

![[phishing-infra.png]]

The primary focus of this blog is on the SMTP relay. In essence, an SMTP relay is  server that 'relays' your email from your own mail server and queues it up for delivery to its final destination. The relay uses the domain name in the email address and DNS to figure out where the email should be sent. 

![[smtp-relaying.png]]

You can configure your own SMTP relay or use a third party relay such as Sendmail, SMTP2GO, Sendgrid, Mailjet or Mailgun. 

### Standalone SMTP Relay Setup
A standalone SMTP relay is completely under our control and acts as a redirector between our target and backend server. Before you set up your SMTP relay, ensure that your backend server (the phishing originator) is fully functional using the checklist above. 

##### Prerequisites
1. Launch a VPS from your provider and make sure it isn't blacklisted. You can use [mxtoolbox](https://mxtoolbox.com/blacklists.aspx) to check whether your domain or IP address is listed against any email blacklists. 
2. SSH into your VPS and configure your IPTables/firewall rules to allow inbound and outbound SMTP traffic. 
3. Configure the correct DNS settings. Basically, just have 2 A records, the first one has a host entry of `@` and the other one is `mail` and both have the SMTP relay's IP address. Although this is optional, you can chooose to have another A record, which has the host entry of `www` and has the IP address of your phishing server.

![[dns-record.png]]

Don't forget the PTR record which is set separately according to your VPS providers instructions. 

After that's setup, we'll start with installing and configuring postfix. (I'll use a regular Ubuntu 18.04 server, but make sure you use the right commands for your server).

 ```shell
 apt-get install postfix
 ```
 
When the dialog pops us, select *Internet Site* then enter the domain name assigned to this VPS. Let's say our VPS's domain name is abc.com, then the system mail name is abc.com

![[postfix1.png]]

![[postfix2.png]]

Then we'll make it act as a redirector by appending this line at the end of Postfix's configuration file which is usually at */etc/postfix/main.cf*.
 ```shell
 postconf -e 'mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128 [BACKEND-SERVER-IP]'
 ```
 
Now that your relay is configured, add it as a relay on your backend server using:
```shell
postconf -e 'relayhost = [RELAY-IP]:25'
```

Then restart Postfix. 
```shell
service postfix restart
```

From my experience, I found that even after setting this up and ensuring that emails were being relayed. They still ended up in the spam folder on Gmail and didn't make it at all through Outlook. Enter third party SMTP relays. 

### Third Party SMTP Relays - Mailjet
There is wide range of third party SMTP relays such as SendMail, SMTP2Go, Sendgrid, Mailgun and Mailjet. While all these options have their own advantages, I chose to use Mailjet as:
- There's no requirement to enter your credit card details when setting up your account
- It has a free quota of 200 emails per day
- It's comparatively a lot easier to setup.

Once you've setup your account on Mailjet, we have a few things to do in order to fully set up our SMTP relay to relay through Mailjet. These options are under the Quick Setup on your dashboard.

![[quicksetup.png]]
- Setup SMTP 
- Manage sender addresses/domains
- Setup domain authentication

#### 1. Managing Sender Addresses
We'll setup our domains before we get into setting up SMTP for relaying. 

Under the **Sender domains & Addresses** page, you'll click on **Add Domain**. A new page will load which allows you to add the domain whose traffic you want to relay. In my case it will be *abc.com*. Ensure that you have complete control over your domain name registrar and in my case it's Namecheap.

![[adddomain.png]]

Press Continue. The next page requires you to validate your domain. There are 2 options available. If you do choose to host a temporary file ensure it is on the home folder of any user, including root. I prefer the DNS method as it's easy and resolves really quick. 

![[validatedomain.png]]

On Namecheap, under Advanced DNS on your domain, create a new TXT record, and assign the values of Host and Value as indicated then set TTL to 5 minutes then save. It should look like this:

![[nc1.png]]

Click **Check Now** and once the record has successfully resolves, you'll get a success message. 

![[success.png]]

#### 2. Setup Domain Authentication

Next, we have to authenticate our domain by setting up SPF and DKIM keys. By default, SPF status and DKIM status are both in error. Click manage button and follow the instructions to add SPF and DKIM records.

**Setting Up the SPF Record**
There are two main points to know about the SPF records:
- SPF record is a TXT record; not be confused with the SPF type. (Although the SPF type could be used, it is not recommended in the industry.)
- There is only one SPF record per domain. If you have more than one SPF DNS record, ISPs will not know which one to use which may cause authentication issues.

![[spf.png]]

**Setting up the DKIM record**
To setup DKIM authentication, you will be creating a new DKIM record. (Unlike SPF records, there are no issues with having multiple DKIM DNS records in your domain.)

![[dkim.png]]

Finally, your records should look like this:

![[final.png]]

Ensure that you confirm the status of your DKIM and SPF records after you've set them up. 

#### 3. Setting Up SMTP
Mailjet issues you with the SMTP server address and SMTP credential which is a set of an API-Key and a password. 

![[api.png]]

Back on our SMTP relay server, we'll edit the Postfix configuration file to set the value of relayhost and setup security and authentication support for SMTP through the use of SASL. 

```bash
sudo nano /etc/postfix/main.cf
```

Then add the following lines to the end of this file.
```bash
relayhost = in-v3.mailjet.com:587
smtp_sasl_auth_enable = yes
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd
smtp_sasl_security_options = noanonymous
smtp_tls_security_level = may
header_size_limit = 4096000
```

Basically:
- The *relayhost* setting forces the Postfix SMTP to send all remote messages to the specified mail server instead of trying to deliver them directly to their destination.
- The *smtp_sasl_auth_enable* setting enables client-side authentication. 
- With the *smtp_sasl_password_maps* parameter, we configure the Postfix SMTP client to send username and password information to the mail gateway server. The username and password are stored in a table that contains one username/password combination for each mail gateway server. In our case, it will be stored in /etc/postfix/sasl_passwd.
- The *smtp_sasl_security_options* disallows anonymous authentication.
- The *smtp_tls_security_level* setting ensures that the connection to the remote smtp server may be encrypted. To explitictly enforce the use of TLS, you can use the *encrypt* value. 
- The *header_size_limit* defines the maximum amount of memory in bytes for storing a message header. If a header is larger, the excess is discarded. By default the value is 102400. 

Then save and close the file. 

Then create the /etc/postfix/sasl_passwd file. 
```bash
sudo nano /etc/postfix/main.cf
```

Add the SMTP relay host and SMTP credentials to this file like below. Replace api-key/username and secret-key with your real Mailjet API key and secret key.
```bash
in-v3.mailjet.com:587 api-key:secret-key
```

Then save and close the file. 

Next, you'll create the corresponding hash db file with postmap.
```bash
sudo postmap /etc/postfix/sasl\_passwd
```

Now you should have a file /etc/postfix/sasl_passwd.db. Restart Postfix for the changes to take effect.
```bash
sudo systemctl restart postfix
```

By default, sasl_passwd and sasl_passwd.db file can be read by any user on the server.  Change the permission to 600 so only root can read and write to these two files.
```bash
sudo chmod 0600 /etc/postfix/sasl_passwd /etc/postfix/sasl_passwd.db
```

At this point, Postfix will send emails via Mailjet directly to inbox on both Gmail and Outlook. Additionally, when you look at the mail headers they'll reveal that the mail originates from mailjet. 

![[inbox.png]]

##### Additional Resources
https://www.socketlabs.com/blog/smtp-relay/
https://www.ired.team/offensive-security/red-team-infrastructure/smtp
https://godlikesecurity.com/index.php/2017/12/14/building-resilient-phishing-campaigns/
https://www.linuxbabe.com/mail-server/postfix-smtp-relay