---
layout: post
title:  "How to setup email in linux"
description: An introduction to memory consistency models.
img:
date: 2018-04-04  +0648
---

There are a multitude of options out there - main of which is [msmtp](https://wiki.archlinux.org/index.php/Msmtp) which can be used with a higher level mailing program like mutt or sendmail or standalone as well.

Sendmail 

First, we configure the `~/.msmtprc` file.

A typical configuration looks like follows:

```
# Set default values for all following accounts.
defaults
auth           on
tls            on
tls_trust_file /etc/ssl/certs/ca-certificates.crt
logfile        ~/.msmtp.log

# Gmail
account        gmail
host           smtp.gmail.com
port           587
from           username@gmail.com
user           username
password       plain-text-password

# A freemail service
account        freemail
host           smtp.freemail.example
from           joe_smith@freemail.example
...

# Set a default account
account default : gmail
```

After this, you can send a mail as 

```
$ echo "hello there username." | msmtp -a default username@domain.com
```

Otherwise, you can add the mail contents in a text file such as:

```
To: username@domain.com
From: username@gmail.com
Subject: A test

Hello there.

```

and send the mail as:

```
$ cat test.mail | msmtp -a default <username>@domain.com
```

I have not been able to get it to work without the -a default set, even though the config file includes the default mail account.

## Protecting password
The previous version has the password written openly in the config file. We can encrypt the password using gpg


