---
title: "No, I didn't try to break Companies House"
date: 2016-12-30T17:00:19+01:00
---

I have a tendency to take a joke too far.

The most recent example of this would be yesterday in IRC, where I was discussing with a few friends what I should name my new company. Being a business to business consultancy, and mostly a one-man operation, the name isn’t that important, but I wanted something with a bit of personality. And I asked:

```
<Mopman> Is there anything stopping me from registering ; DROP TABLE
"Companies";-- Ltd as a company name?
```

We weren’t sure of the answer. So I gave it a go. Turns out, the answer is yes you can, that is a perfectly legitimate company name. Also turns out, if you do that, you’ll attract the attention of people around the world, even if you don’t tell anyone other than a few close friends about your new venture.

I’m really happy that many people seem to like my company name! I like it too, even more now than I did when I registered it. And if you need any application security work done, give me a shout - especially if you’re a non-profit, because I like you guys.

But I need to clear something up: *I did not attempt to break Companies House*!

The company name is a bit of hacker sleight-of-hand… or as some astute people have put it, it’s ‘wrong’. Of course it’s wrong - I’m not a /total/ arsehole. :)

For those not intimately familiar with SQL injection, it occurs when user input is concatenated with an SQL query insecurely and an attacker is able to modify the query to do things it wasn’t supposed to do. But my input doesn’t do that. It just looks (on the surface) like it might.

The example many are comparing it to is the infamous Bobby Tables from XKCD. The input used in Bobby Tables is:

```
Robert'); DROP TABLE Students;--
```

The SQL query that goes along with this on the server would be something like:

```
SELECT * FROM Students WHERE (Name = '<user input goes here>');
```

So by entering the comic’s input, you get:

```
SELECT * FROM Students WHERE (Name = 'Robert'); DROP TABLE STUDENTS;--';
```

The ‘); after Robert closes the preceding part of the query, and then the attacker inserts their own query. The – at the end is a comment, and makes SQL ignore everything after this point.

However, my company contains nothing to close the preceding part of the query. If you insert my company name here, you get:

```
SELECT * FROM Students WHERE NAME = '; DROP TABLE "Companies";-- Ltd';
```

The whole ‘invalid’ input is actually valid input, inside the string as expected - it just looks like SQL. This is not going to cause a problem. Even with bad code. Everything in red above is just treated like a normal text string.

In addition, as many have pointed out, most SQL engines don’t accept double quotes around table names - so even IF this was treated as a query (which it will not be), it would likely fail to drop anyway. Call that ‘defense in depth’ if you like.

TLDR: It’s pretend. Just a joke! But a funny one, apparently. :)

EDIT:

Yes, this is a simplified example, and there are many many ways this string could be concatenated in various codebases. However, there is none, outside really whacky examples which would be breaking on many other company names already, where it causes a major issue.
