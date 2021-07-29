+++ 
date = 2021-07-29T20:33:37+01:00
title = "Adventures in PHP Static Analysis with Psalm"
images = ["images/psalm.png"]
+++

I learned recently of [Psalm](https://psalm.dev/), the open source static
analysis tool for PHP released by Vimeo. And then I did a happy dance, because I've
been looking for a good free SAST tool for PHP for a while.

So the next thing I wanted to discover - is it any help for finding vulnerabilities? And
the answer looks to be yes. It has a taint analysis mode, is configurable
with custom sinks/sources via plugins and can output the taint graph
for us as well as reports in SARIF for other tools to consume (ie. GitHub). Neat.

The Psalm devs are eagerly open with the fact that the taint analysis mode of
Psalm isn't considered batteries-included - actually they use the metaphor of
a truck with half a tank of gas, waiting for us to supply the other
half - which sounds perfect, so in this post I will cover adding a little gas, to
detect a vulnerability class that it doesn't detect by default.

### Part One: Our Exploitable Test Case

As [might be clear](https://pizzey.me/posts/exploiting-an-unexploitable-squirrelmail-bug/),
I really like deserialisation issues. Everybody needs a hobby. Out of the box, Psalm will tell us if we try to do something obviously silly, like
pass a user-controlled string directly to ``unserialize()``:

```
sam@DESKTOP:~/sauce/test$ ./vendor/bin/psalm --taint-analysis

ERROR: TaintedUnserialize - index.php:14:13 - Detected tainted code passed to unserialize or similar (see https://psalm.dev/250)

  $_GET['x'] - index.php:14:13
unserialize($_GET['x']);
```

But, what it doesn't realise is that in
PHP, [many methods in the standard library that appear safe are equivalent to
unserialize()](https://i.blackhat.com/us-18/Thu-August-9/us-18-Thomas-Its-A-PHP-Unserialization-Vulnerability-Jim-But-Not-As-We-Know-It.pdf), 
as long as the attacker can control the beginning of the string,
and can put a file they control onto the filesystem. This is
because many innocuous-looking file functions in PHP can accept
[stream wrappers](https://www.php.net/manual/en/wrappers.php) in place of filenames,
and one of the included stream wrappers, 'phar://', will unserialize part of the
file you point it to upon loading it. This behaviour is included in PHP by default,
cannot be easily disabled, and occurs on functions like ``file_exists()``, which most
developers would not consider to be a dangerous function - for example see [CVE-2018-19296](https://github.com/advisories/GHSA-7w4p-72j7-v7c2)
in PHPMailer's ``addAttachment()`` function.

This behaviour is, as far as I know (further testing is on my todo list), fixed in PHP 8.
However, nobody upgrades PHP, so we should be able to pop shells with it for a while longer. :)

So, in order to reproduce this issue and test our plugin later, I created a
vulnerable test case like so:

{{< highlight php >}}
<?php
// Psalm doesn't treat $argv as a taint source by default so we sneak it into
// the $_GET superglobal here just for testing purposes.
$_GET['x'] = $argv[1];

// This is our 'POP chain' of sorts. This should under normal conditions never
// be executed, only if we managed to exploit the rest of the script.
class BangBang {
  function __destruct() { die("I hit the ground...\n"); }
}

// Now we perform a normal file operation with a phar:// stream on exploit.phar
// which results in a deserialize of the above: `php test.php "phar://exploit.phar"`
echo "The owner of your file is: " . fileowner($_GET['x']) . "\n";
{{< / highlight >}}

Running this with a normal filename as the argument as expected, we get the
expected output:

```
sam@DESKTOP:~/sauce/test$ php index.php composer.json
The owner of your file is: 1000
```

However if we run this by passing a phar:// stream wrapper, to a specially
created .phar file (creating this can be an exercise for the reader), this 
happens:

```
sam@DESKTOP:~/sauce/test$ php index.php phar://exploit.phar
The owner of your file is: 0
I hit the ground...
```

To recap what is happening here as we're going quite fast:
* When we call ``fileowner("phar://exploit.phar")``, PHP attempts to open exploit.phar as a
  PHAR archive due to the stream wrapper in the filename.
* It deserializes part of the PHAR archive metadata, which is attacker-provided. This
  instantiates an instance of the ``BangBang`` class.
* When our ``BangBang`` instance falls out of scope, the destructor is called. In this case,
  it prints a message, but, our attacker controls all properties of this object, which
  enables more complicated exploits via
  [Property-Oriented Programming](https://vickieli.medium.com/diving-into-unserialize-pop-chains-35bc1141b69a).

Just to note, our file doesn't need to end in .phar, PHP will attempt to load a file with any extension. So
getting the file onto the filesystem can be achieved via normal functionality of the application
that allows for uploading images, etc. assuming we can know the path to the uploaded file.

By default, Psalm does not see anything wrong with our test script, even though it's exploitable:

```
sam@DESKTOP:~/sauce/test$ ./vendor/bin/psalm --taint-analysis
Scanning files...
Analyzing files...

░
------------------------------
No errors found!
------------------------------

Checks took 0.85 seconds and used 63.333MB of memory
```

So let's fix that.

### Part Two: Adding A Sink To Psalm

If you are not familiar with taint analysis, the above output from Psalm might
seem mysterious. But it's a simple concept. Taint analysis tools
operate on the concept of sources and sinks - sources are places in a codebase
where data may be tainted - that is, user-provided. And sinks are the locations
in a codebase where tainted data should not reach. Think of for example SQL
injection - you never want a source, such as the query string of a URL, to
reach a sink, such as ``mysql_query()`` - at least without being untainted first.

So, Psalm builds a graph of all the ways data can flow through the
application from sources to sinks, and if any nodes connect, it warns us. It will
follow this flow even if the variable is assigned to a new one, concatenated,
and so on, unless we tell it the data is untainted now - for example if we passed it
through a function that we tell it makes it safe again.

So in order to have Psalm detect our vulnerability, we just need to add all of the
functions that accept stream wrappers as sinks. The
[documentation](https://psalm.dev/docs/security_analysis/custom_taint_sinks/) shows
us that we can do this with an Annotation on the function, which seems to be
a problem as we can't annotate a standard library function. After looking at how
Psalm itself handles this, it turns out the appropriate thing to do is stub this
function out as a normal user defined function and annotate that. Normally we
can't do this in PHP, as you can't redefine a function, but it makes
sense here as we're not actually executing the code, just telling Psalm about it.

It turns out this only works if it's in a plugin. I'm not sure if this is intentional
behaviour in Psalm or a bug - it's OK in this case, as we would like to reuse this
in multiple projects anyway. So, I created a plugin for Psalm,
[funserialize](https://github.com/pizzeys/funserialize). You can check the Psalm
documentation to see how to create a plugin, but basically the only relevant file
here is [stubs/funserialize.phpstub](https://github.com/pizzeys/funserialize/blob/main/stubs/funserialize.phpstub) - 
and all this does is stub out the functions we are interested in catching uses of, and annotate them as sinks for Psalm, like this:

{{< highlight php >}}
/**
 * @psalm-taint-sink file $filename
 */
function fileowner($filename) { }
{{< / highlight >}}

I included every function that accepts stream wrappers, as far as I know, that
isn't already supported by Psalm for other reasons. If we add this plugin to
our composer.json and then enable it with
`./vendor/bin/psalm-plugin enable Mopman\\Funserialize\\Plugin`, and *then*
run Psalm, it knows that our program is vulnerable now:

```
sam@DESKTOP:~/sauce/test$ ./vendor/bin/psalm --taint-analysis
Scanning files...
Analyzing files...

░

ERROR: TaintedFile - index.php:15:48 - Detected tainted file handling (see https://psalm.dev/255)
  $_GET
    <no known location>

  $_GET['x'] - index.php:15:48
echo "The owner of your file is: " . fileowner($_GET['x']) . "\n";

  call to fileowner - index.php:15:48
echo "The owner of your file is: " . fileowner($_GET['x']) . "\n";
```

Excellent! I wonder... would it have found a real bug?

### Part Three: Patting Ourselves On The Back Testing

As Totally Professional Security Researchers, we should backtest our tool changes to make
sure they catch historical bugs. Or to put it another way, I spent all this time figuring out
how this works, and now I want to see it find a real bug instead of a tiny testcase.

So, we shall take a look at the PHPMailer issue again. First, we check out the vulnerable
version of PHPMailer and set up Psalm:

```
sam@DESKTOP:~/sauce$ git clone https://github.com/PHPMailer/PHPMailer
sam@DESKTOP:~/sauce/PHPMailer$ git checkout v6.0.5
sam@DESKTOP:~/sauce/PHPMailer$ composer require vimeo/psalm
sam@DESKTOP:~/sauce/PHPMailer$ ./vendor/bin/psalm --init
```

As PHPMailer is a library, we need to include some code that actually uses the
library in a vulnerable way - this would usually be in the application itself,
but for this case I place the following in ``src/index.php``, simulating what
an unsuspecting application might do:

{{< highlight php >}}
<?php

require_once "PHPMailer.php";

use PHPMailer\PHPMailer\PHPMailer;

$mail = new PHPMailer();
$mail->addAttachment($_GET['filename']);
{{< / highlight >}}

And then run Psalm:

```
sam@DESKTOP:~/sauce/PHPMailer$ ./vendor/bin/psalm --taint-analysis
```

And here is where I'd love to tell you it found nothing, because it would make for a cleaner
example. But I didn't pick my bugs carefully enough until this point, and it turns out
Psalm detects this issue anyway, because it detects it on the ``file_get_contents()`` sink,
which they already have detected for file disclosure reasons. :)

But, we can still test this, by commenting out the *other* vulnerability on line 3232 of
PHPMailer.php, and then running Psalm again to ensure it runs clean.

Then if we enable Funserialize and run again:

```
sam@DESKTOP:~/sauce/PHPMailer$ composer require mopman/funserialize @dev
sam@DESKTOP:~/sauce/PHPMailer$ ./vendor/bin/psalm-plugin enable Mopman\\Funserialize\\Plugin
sam@DESKTOP:~/sauce/PHPMailer$ ./vendor/bin/psalm --taint-analysis

ERROR: TaintedFile - src/PHPMailer.php:1825:33 - Detected tainted file handling (see https://psalm.dev/255)
  $_GET
    <no known location>

  $_GET['filename'] - src/index.php:8:22
$mail->addAttachment($_GET['filename']);

  /* snipped here for brevity */

  call to file_exists - src/PHPMailer.php:1825:33
        $readable = file_exists($path);
```

It now finds our problematic calls to ``file_exists()`` and ``is_readable()``. Nice.

### Conclusion

I can definitely see Psalm becoming a regularly used tool for speeding up analysis of PHP applications.

I think the particular sinks added here should probably be categorised as something else other than file manipulation,
but I don't know how best to do that yet and plan to investigate more deeply, which is why it's still a plugin
and not submitted to the main repo itself until I understand more. Hopefully Psalm can handle the
'not applicable to PHP 8' issue and I can try to get them in as equivalent to ``unserialize()``.

There's definitely scope for improvement in Psalm itself, particularly with performance, on larger
codebases my aging machine has *struggled* with RAM at times, and on Wordpress I had to actually remove some code
to get it to even finish (which is why this example isn't using Wordpress!). But overall it runs smoothly and
is fantastically easy to use, so you should definitely check it out.
