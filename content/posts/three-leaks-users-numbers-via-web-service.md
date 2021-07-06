---
title: "Three leaks users numbers via web service"
date: 2014-03-18T16:48:50+01:00
---

OK, so you can file this one under "annoying, but not the end of the
world".

Three, like most mobile providers, has a mobile application that comes installed
on your phone if you buy one from them, or you can download separately if you
wish. It mostly replicates a limited subset of the MyThree website, allowing
you to view your recent invoices, whether you are eligible for an upgrade, etc.

The first thing that jumped at me when I used this is that it has no authentication
whatsoever - they just know who you are. This isn't particularly interesting - the
app only works on 3G and not when connected to Wifi, so it's using the headers that
get attached to many mobile networks requests for "special" hosts - you may remember
these as [the ones O2 famously leaked to the world once](http://www.theregister.co.uk/2012/01/25/o2_hands_out_phone_numbers_to_websites).

No problems so far, except that they leak those headers to any application on the
system who wants them - if it has internet permissions, but not READ_PHONE_STATE, it
can just hit:

```
http://mobile.three.co.uk/om_services/threeuserinfo/headers?app_id==ThreeAppAndroid
```

Which returns an XML file looking something like below:

```
<X-H3G-DEVICE-NAME>LG-Nexus-5</X-H3G-DEVICE-NAME>
<X-H3G-PARTY-ID>xxxxxxxx</X-H3G-PARTY-ID>
<X-H3G-ACCT-DATA>xxxxxxxx</X-H3G-ACCT-DATA>
<X-H3G-MSISDN>xxxxxxxxxxxx</X-H3G-MSISDN>
```

I don't know what "party id" and "account data" are. I'm curious if it's the same
as my Three account number on my bill, but MyThree is down at the moment, so I
can't check.

However, if you have a popular android app and need some targetted phone numbers to spam,
that's what MSISDN is. Of course, users don't actually check permissions,
so you could just request READ_PHONE_STATE with no justification anyway.

Makes you wonder what other fun stuff might be on their services though, eh.
