## Mailcow: Setting up a full featured self hosted mail server

> Setting up a full featured mail server is an actual **PAIN** for a system admin, you have to interconnect many software pieces and a lot of testing espacially security and spam if you plan to use it for your company ... In this Post i will take you in my journey of setting up  a  full featured  mail server, i was searching a lot on the internet and i didn't get a sweetable answer to move on .. but finally i successfully made it ! .. It works with multiple domains, secure and no spam problems !


# For the reader 

**I assume you have knowledge about how email works, Mail DNS records, linux, docker and SSL of course. This guide is not for beginners**

If you dont have enough knowledge, take a seat and take a look here : 

|Term|Definition|
|:-:|:-:|
|**Mail server**|Is a computer system that sends and receives email [_source_](https://techterms.com/definition/mail_server)|
 |**MX**| Used to tell the world which mail servers accept incoming mail for your domain [_source_](https://help.dreamhost.com/hc/en-us/articles/215032408-What-is-an-MX-record-)|
|**CNAME**|Used to alias one name to another,CNAME stands for Canonical Name. [_source_](https://support.dnsimple.com/articles/cname-record/) |
|**DKIM**|Is an email authentication technique that allows the receiver to check that an email was indeed sent and authorized by the owner of that domain [_source_](https://www.dmarcanalyzer.com/dkim/) |
|**SPF**|Is an email-authentication technique which is used to prevent spammers from sending messages on behalf of your domain  [_source_](https://www.dmarcanalyzer.com/spf/)|
|**DMARC**|is an email validation system designed to protect your company’s email domain from being used for email spoofing [_source_](https://www.dmarcanalyzer.com/dmarc/) |

# Mailcow 

![256.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1610109314178/O0QJMagRl.png)
Mailcow is a free, open source software project. A Mailcow server is a collection of Docker containers running different mail server applications, SOGo, Postfix, Dovecot etc. Mailcow provides a modern and easy to use web interface to create and manage email accounts. You can visit the [official documentation](https://mailcow.github.io/mailcow-dockerized-docs/)

I ill not make a comparison about different opensource self hosted mail servers, rather then that i will give you a great repo that includes owesome softwares marked as self hosted, you can check the Email  section  [come here](https://github.com/awesome-selfhosted/awesome-selfhosted#complete-solutions)

If you want some thoughts about mailcow from real users check this reddit discussions : [come here](https://www.reddit.com/r/selfhosted/comments/hdgau0/whats_the_current_take_on_selfhosting_mail/) and [here](https://www.reddit.com/r/selfhosted/comments/4ct285/iredmail_vs_mailcow_vs_mailinabox/)

# Server Requirements 

The recommended OS to run mailcow is `ubuntu 18.04`, don't use cento7 because the maintainers said : 
> Do not use CentOS 8 with Centos 7 Docker packages. You may create an open relay.

For the resources i bought a VPS from OVH cloud provider with : 
* 2 VCPU 
* 4 GB of RAM 
* 80 GB storage 

Docker installed : 
```bash
curl -sSL https://get.docker.com/ | CHANNEL=stable sh
systemctl enable docker.service
systemctl start docker.service
```
Docker-compose installed : 
```bash 
curl -L https://github.com/docker/compose/releases/download/$(curl -Ls https://www.servercow.de/docker-compose/latest.php)/docker-compose-$(uname -s)-$(uname -m) > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

If you have a firewall, you should allow those ports : 
```bash 
netstat -tulpn | grep -E -w '25|80|110|143|443|465|587|993|995|4190'
```

> **Note** : you must have a clean server means no other applications or a reverse proxy becaue mailcow has every thing in place.

Okay now every thing is good ! but there is another thing to verify which is our IP !!! 
we have to test it if it is **blacklisted**, if yes it's a huge problem ... we can't proceed anymore because your  mails will not be delivered, it's all about your host **reputation**!. 

We can check it with an online service called **mxtoolbox** by visiting this url [mxtoolbox](https://mxtoolbox.com/SuperTool.aspx).

# Basic DNS configuration

Before we proceed we have to get a domain name from any provider, i got one from `OVH`, and i delegated to `AZURE DNS SERVICE`, it dosen't matter actually, we will have the same configuration in any provider.

I pretend that i have : 
* **Root Domain name** : mymailserver.com
* **Server IP address** : 1.2.3.4 

**Let's get started !**

 ## Create an **A** record 

In your provider DNS pannel add an `A` record like this 

* **Name** : mail 
* **Type** : A
* **TTL** : default 
* **Value** : 1.2.3.4

You can confirm with the `dig` command : 
```bash 
 dig mail.mymailserver.com +noall +answer
```
## Create the **CNAME** records

In your provider DNS pannel add a `CNAME` record for the `autoconfig` like this 

* **Name** : autoconfig 
* **Type** : CNAME
* **TTL** : default 
* **Alias** : mail.mymailserver.com

Test it with : `dig autoconfig.mymailserver.com +noall +answer`

Add another `CNAME` record for the `autodiscover` like this :
* **Name** : autodiscover 
* **Type** : CNAME
* **TTL** : default 
* **Alias** : mail.mymailserver.com

Test it with : `dig autodiscover.mymailserver.com +noall +answer`

## Create the an MX record 

In your root domain add an MX domain to point to your mail server domain : 

* **Name** : @/empty  
* **Type** : MX
* **TTL** : default 
* **Prefenerence/periorty** : 10 or 0 
* **Mail Exchange** : mail.mymailserver.com

## rDNS configuration 

Get more information about `rDNS` [here](https://help.returnpath.com/hc/en-us/articles/360019088671-How-to-set-up-reverse-DNS-zone-and-PTR-records-) 

You can configure your rDNS on the provider of your `server` and change the generated `domain name` to your mail domain `mail.mymailserver.com`

and test it with `dig -x 1.2.3.4 +noall +answer`, should get `mail.mymailserver.com` as responce.

# Security DNS Records 

Authenticate your mail server and protect it from [Fake identities](https://www.serversmtp.com/what-is-dkim/)  and  [Domain spoofing attacks](https://www.barracuda.com/glossary/domain-spoofing) we have to setup those records 

## SPF record 

In your root domain add a `TXT` record like this : 

* **Name** : @/empty  
* **Type** : TXT
* **TTL** : default 
* **value** : v=spf1 ip4:1.2.3.4 -all

Test it with `dig mymailserver.com TXT  `

## DKIM record 

Setting up the dkim record needs a public key to be inserted in the record here but we can't now because we have to install mailcow and get the public key given by our mail server, we will leave it empty for now. 

* **Name** : dkim._domainkey
* **Type** : TXT
* **TTL** : default 
* **value** : v=DKIM1;k=rsa;t=s;s=email;**p=<private-key>**

## DMARC record 

The best thing you can do for dmarc is to make a free account on [dmarcian](https://dmarcian.com/), it will give you a record like this : 
```bash 
v=DMARC1; p=reject; rua=mailto:<dmarc email1>; ruf=<dmarc email2>
```
Just add it to your domain DNS pannel like this : 

* **Name** : _dmarc
* **TTL** : default 
* **value** : v=DMARC1; p=reject; rua=mailto: dmarc email1 ; ruf=dmarc email2

Test is with : `dig _dmarc.mymailserver.com TXT`


**That's it for the DNS configurations, all records are in place execpt the dkim value we ill add it later**

# Install mailcow 

You need `git` installed to clone the repo in your server : 

```bash
 git clone https://github.com/mailcow/mailcow-dockerized
 cd mailcow-dockerized
```

Your in the repo now, so run the script `./generate_config.sh
` to generate your config, for the hostname add `mail.mymailserver.com`, it will request a certificate from `Let'svEncrypt` automatically. 

To run your mail server just use this command : 

```bash 
docker-compose pull
docker-compose up -d 
```

**You need to wait some time to download all the containers, run them and provision a SSL certificate.**

The default credentials to access are `admin` & `moohoo` (you must change it with your own)

Verify that all containers are healthy with the `green icon` like this :


![Screenshot_2021-01-08 mailcow UI.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1610120906694/XD99OD4X3.png)

**Well done ! Let's proceed to add domains and mailboxes**


# Configure Mailcow 

Our mail server is working now but there is neither domains or mailboxs configured. in this section we will go step by step .. let's GO !

## Add your domain 

Login to your mailcow admin panel :

1. Select `configuration` on the top bar then select `mail setup`
2. Click on `add domain` then add your root domain `mymailserver.com` and click on `Add domain and restart SoGo`
3. Select `configuration` again then click on `configuration & details`
4. In the hosizontal menu click `configuration` and click on `ARC/DKIM keys`
5. Scroll to bottom and fill the `domain/s` form with your domain `mymailserver.com`
6. On the selector type `dkim` 
7. Select `2048` on the `DKIM key length` and click `add`
8. Copy the generated public key that starts with `v=DKIM1;k=rsa;t=s;s=email;p= ...`
9. Go to your DNS pannel and modify the `TXT` record with the generated value
10. Wait until your DNS modification propagate  

You can validate your configuration for `SPF`, `DKIM` and `DMARC` with this        [site](https://dmarcian.com/), you should get result like this (don't forget to put `dkim` as a selector` :

![Screenshot_2021-01-08 Free DMARC Domain Check Is Your Domain Protected - dmarcian.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1610122571096/eV1_YdWK8.png)

## Create a Mailbox for mymailserver.com

By default mailcow give `10GB` of storage to every domain and every mailbox under that domain has `3GB` of storage .. so feel free to modify them to your needs, 

To create a mailbox folow those steps : 

1. Under `configuration` on the to bar click `mail setup` and select mailboxes`
2. Click `add mailbox`
3. Choose a `username` , your domain (it will be loaded automatically) and then add a password and click `add`
4. On the top bar click `apps` and then `webmail`, you will be redirected to the `SoGo` UI , login wiyh your newly created mail and password
5. on the bottom click on the green icon, you will get an interface of sending a mail 
6. In your browser open another tab and go to this site : mail-tester.com and copy the mail address.
7. Return to your `SoGo` interface and send a mail with that mail, on the body make sure to add some text with a least two paragraphs.
8. Wait 10 seconds and go the mail-testor.com and click on `then check your score` 

And **BOOM** you should get this result **You Can Send**  : 


![Screenshot_2021-01-07 Spam Test Result.png](https://cdn.hashnode.com/res/hashnode/image/upload/v1610123568730/QfI2NwVUH.png)



Feel free to send a mail to your friends and your own Gmail address, don't worry Gmail still don't know your mail server, so he will placed as spam, just `unspam` it !, try to send it to zoho mail for example. The test again with other tools i suggest to use : 

* [Mxtoolbox SuperTool](https://mxtoolbox.com/SuperTool.aspx) and select `test mail server`
* [Dkimvalidator](https://dkimvalidator.com/) and send an email to the given address, make sure that all tests are oassed like the `dmarcian.com` tests.

## Add another domain to mailcow 
I assume we have a second domain : myseconddomain.com.
I will not repeat the same steps it's quite easy so let's go :

1. Create A record : `mail.myseconddomain.com` that points to your Host `1.2.3.4` 
2. Create a CNAME record : `autoconfig` that points to `mail.myseconddomain.com`
3. Create a CNAME record : `autodiscover` that points to `mail.myseconddomain.com`
4. Create an MX record: on the root domain add `10` as periority and `mail.myseconddomain.com` as target 
5. Create SPF record like we did earlier
6. Create DMARC record as we did earlier 
7. Create DKIM record and in the same time add your new domain as we did earlier and copy the generated `DKIM key` to your `DKIM` record. 
8. Validate your records 
9. Add a mailbox under your new domain and send an email to mail-tester.com and dkimvalidator.com, you should get 10/10 sweetheart :)

# Final thoughts 

Don't send a lot of mails directly you will be blocked ! ... so start step by step  by sending 20 mails per day for the first week then try sending 80 the next week ... after 2 months you can send a lot of mails like 1000 mails, but take in mind that you need to add the `list unsubscribe headers` to your postfix to allow users to unsubscribe against your newsletters/subscriptions. 





