---
title: "Protecting from the CPanel Ransomware That Doesn't Exist Yet"
date: 2017-01-15T17:11:33+01:00
---

Ransomware just ain’t what it used to be. From the now-relatively-Paleolithic Cryptolocker until today we’ve seen the criminals who run these operations evolve to meet the responses brought against them - from [offering free decryption if you infect others](https://www.wired.com/2016/12/popcorn-time-ransomware/), to [exploiting insecure MongoDB instances](http://www.computerworld.com/article/3157766/linux/mongodb-ransomware-attacks-and-lessons-learned.html) they have shown they’re totally willing to find new avenues to get an edge in this lucrative business.

One thing we haven’t seen a lot of though is ransomware targeting website hosting environments. And it makes a lot of sense - if they target your generic Linux hosting environment customers, they are hitting core business infrastructure that the business needs to operate. It’s also probably a smaller/mid-size business, or at least one without a technical focus, and so the victim is likely to not have the infrastructure to recover from it without paying you. And on top of that, losing your website is embarassing, and everyone can see the effects, hastening the payment.

Sure, we’ve seen a few, half-assed attempts - Sophos has a writeup on something they call [PHPRansm-B](https://nakedsecurity.sophos.com/2016/03/02/php-ransomware-attacks-blogs-websites-content-managers-and-more/) (which I’d like a sample of if anyone could pass me one, by the way), but I am fairly convinced that at some point we will see a devastatingly effective version of this, and we should probably be preparing now. So how would it look, and what are the risks?

Well firstly, we cannot prevent people from getting infected by it. Sure, we can try. We can throw WAF’s and maldet scans up and tell people to update their software until the cows come home. But they will still get hit via unpatched software and bad passwords. They do now, all the time, despite our best efforts. Industry ‘secret’ for you: many of our customers would be out of luck every week, if it wasn’t for our backups. They don’t have any of their own, and they’re running a plugin from 2009. The WAF can only do so much.

But we have the backups, right? So, unlike many corporate environments, we are doing the number one thing that helps mitigate ransomware. But how are we doing backups?

For many hosting companies, the answer is ‘with CPanel’ - because they’re already running CPanel, so why not use its built-in backup system? Industry ‘secret’ 2: CPanel’s backup system is not good in this scenario.

For those who’ve never looked into it, CPanel contains a configuration file, [cpbackup-exclude.conf](https://documentation.cpanel.net/display/CKB/How+to+Exclude+Files+From+Backups). The most common use of this is to globally-exclude files from the backups in /etc/cpbackup-exclude.conf, but it can also be used locally from the users home directory. Which our theoretical ransomware has access to.

So, an attacker finds a few 0-days in commonly used software, infects as many as it can, updates this file to exclude ‘*’ (which in my tests, excludes all of public_html and also email!) and waits for 30 days. After the monthly backups have rotated out, it encrypts everything. The user contacts their host for a backup restore, and there is one, it’s totally valid and popped no warnings, but restoring it has no effect, they’re still boned. Oops.

The database will still be there, thankfully. I’ve not managed to find a reliable method of excluding databases silently, but I suspect there is one (I’ve had scenarios before where database backups are mysteriously missing for users, and we had no indication). Even if not, though, losing email&files is probably enough, and the attacker could always get super fancy and encrypt the database on day-1 with a shim that decrypts it on the fly until ransom day, or something.

I fear this non-existent malware. And I would bet money on it coming soon, which is why I don’t feel too bad about giving out the blueprints. What can we do about it now?

Well, the real answer is to stop using a backup system that is compromisable by the attacker. I don’t have any suggestion on which to use (I have gripes and pros for all of them). In the short term, create a cpbackup-exclude.conf for all of your users, and make it immutable. There might be a better way to disable this functionality entirely - does anybody know? I asked around and got nothing. This also isn’t a dig at CPanel particularly, other similar systems have the same problems.

Also, try to herd your users into updating software regularly. Somehow.

Restless dreams!
