---
title: "Never let it be said that I waste time..."
date: 2014-01-07T16:41:54+01:00
---

A friend of mine recently shared the following on Facebook:<!--more-->

```
If: A B C D E F G H I J K L M N O P Q R S T U V W X Y Z
Is equal to;
1 2 3 4 5 6 7 8 9 10 11 12 13 14 15 16 17 18 19 20 21 22 23 24 25 26

Then
H+A+R+D+W+O+R+K ;
8+1+18+4+23+15+18+11 = 98%

K+N+O+W+L+E+D+G+E ;
11+14+15+23+12+5+4+7+5 = 96%

L+U+C+K ;
12+21+3+11 = 47%

None of them makes 100%.
Then what makes 100% ???
Is it Money?
NO! M+O+N+E+Y= 13+15+14+5+25 = 72%

Leadership?
NO! L+E+A+D+E+R+S+H+I+P= 12+5+1+4+5+18+19+8+9+16 = 97%

Every problem has a solution, only if we change our "ATTITUDE".

A+T+T+I+T+U+D+E ;
1+20+20+9+20+21+4+5 = 100%
```

This important discovery, this biological algorithm that could lead us to greatness
when otherwise we felt impotent, spoke to me deeply. So I was inspired to
work further on this and see if I could discover more ways to access its power.
I&rsquo;m not too good at mental arithmetic, so I used the following Ruby code:

```
#/usr/bin/env ruby

def calculate_score(word)
  word.each_char.inject(0) { |score, n| score + n.ord - 64  }
end

IO.foreach('wordsEn.txt') do |l|
  puts l if calculate_score(l.chomp!.upcase) == 100
end
```

I used the great wordlist from here and let it run for a bit. I was able to discover that alongside attitude, an aneurism, crudity, dryrot, fatherhood and, most importantly, liberalism are also able to unlock this newfound perspective on life.

It has been a productive day.
