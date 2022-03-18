+++ 
date = 2022-03-18T08:35:37+00:00
title = "Is it a string? Is it an integer? No, it's a valid admin token!"
images = ["images/superman.png"]
+++

Last autumn, I [reported](https://www.ispconfig.org/blog/ispconfig-3-2-7-released-important-security-update/)
a vulnerability to the ISPConfig team that could allow for authentication bypass on their API. I held off on
writing about it in detail at the time due to the impact - although successfully exploiting it required the
planets to align for the attacker a little, I popped enough shells in the lab to be uncomfortable! But it's
been a while now and hopefully everyone is patched, and the bug is a fun journey.

### The Database Class
I began auditing the ISPConfig codebase by looking at the database layer. This is often the first thing I
look at in a web application that's new to me, partly because by its very nature, it will be receiving
untrusted data at some point, and partly because there are a lot of mistakes a developer can make in a database
layer - SQL injection, obviously, but also very common are type handling issues. A database is expected to process
a variety of data types, and there's bound to be some conversion and/or casting of these types on the way from the
network protocol, to the app, and then to the database - many chances for something
to be given some data in a format it wasn't expecting.

So, let's take a look at an abbreviated version of the function in ISPConfig that builds an SQL query:

{{< highlight php >}}
public function _build_query_string($sQuery = '') {
	$aArgs = func_get_args();

	foreach($aArgs as $sKey => $sValue) {
		$iPos2 = strpos($sQuery, '??', $iPos2);
		$iPos = strpos($sQuery, '?', $iPos);

		if(is_int($sValue) || is_float($sValue)) {
			$sTxt = $sValue;
		} elseif(is_null($sValue) || (is_string($sValue) && (strcmp($sValue, '#NULL#') == 0))) {
			$sTxt = 'NULL';
		} elseif(is_array($sValue)) {
			$sTxt = '';
			foreach($sValue as $sVal) $sTxt .= ',\'' . $this->escape($sVal) . '\'';
			$sTxt = '(' . substr($sTxt, 1) . ')';
			if($sTxt == '()') $sTxt = '(0)';
		} else {
			$sTxt = '\'' . $this->escape($sValue) . '\'';
		}

		$sQuery = substr_replace($sQuery, $sTxt, $iPos, 1);
		$iPos += strlen($sTxt);
		$iPos2 = $iPos;
	}
}
{{< / highlight >}}

I've removed some code here to save space, but the important takeaways are:

* The function takes a variable number of arguments and then inserts them into a provided query.
* It can handle a variety of data types, and has different escaping behaviour for each.
* On the backend, it's not using a prepared statement, merely concatenating a query string.

Initially I was interested in the array behaviour, as this is a common source of bugs for me.
As HTTP is a text protocol, web developers have a tendency to consider
any user-supplied data to always be a string, and will often hook an application that assumes it
only ever gets strings up to a database layer that can handle other types. However, putting an array
into the PHP superglobals is trivial, for example if you pass a query string like `?myArray[]=1&myArray[]=2`,
then `$_GET['myArray']` will be an array containing "1" and "2".

Here though, ISPConfig actually does pretty well, the function above will walk the array and escape the
elements as strings - the worst we can do here is a zero, if we can pass an empty array. This isn't great,
but ISPConfig does a lot of `if(is_empty($param))` checks, which will prevent us from abusing this. I have some
further research ideas here you might be able to guess, but I haven't had time to hit the PHP source yet. :)

However, if you look at the branches that handle integers and floats, you'll notice that those pass their
value into the query directly - without escaping or checking them at all. And now we need to learn something about comparing data
of different types in MySQL...

### MySQL Is Very Weird

Pop quiz time! What happens in MySQL if you compare a string with an integer? If you said "It doesn't let you",
then stay behind after class. If you said "It returns 0", then... partial credit:

```
MariaDB [(none)]> select ("NurGeträumt" = 99);
+----------------------+
| ("NurGeträumt" = 99) |
+----------------------+
|                    0 |
+----------------------+
1 row in set, 1 warning (0.00 sec)

MariaDB [(none)]> select ("99Luftballons" = 99);
+------------------------+
| ("99Luftballons" = 99) |
+------------------------+
|                      1 |
+------------------------+
1 row in set, 1 warning (0.00 sec)
```

MySQL actually converts the string to an integer by taking the leading digits from it, and then casting the
value of those digits to an integer - actually, very similar to the PHP behaviour when using == to compare
a number and a string. So, if we can pass an integer to an application that only expects to receive strings,
we can potentially force a comparison to happen in MySQL that isn't what the application expected.

But we interact with ISPConfig over HTTP, where we can only send strings! How could we pass an integer?

### The REST API

As well as the web interface, ISPConfig provides a REST API which can be used to perform tasks in
an automated fashion. For instance you can hook your billing system up to it, and when a user signs
up to your service and pays you, their account is automatically created over the API. The API can be
sent its data in a number of formats, such as JSON and XML. JSON and XML have true integer types,
freeing us from HTTP's string based shackles.

However, using the API requires that the admin create a "remote user" in the web panel, which is used
to authenticate your requests. Well, that's not quite accurate - it's used to authenticate *one* request,
`login`, which then provides you with a randomly generated session token you then send with further requests.

Then, every subsequent request will do this before doing anything further (via `getSession()`):

{{< highlight php >}}
	// NOTE: $session_id comes from the user-provided XML/JSON and is not necessarily a string...
	$sql = "SELECT * FROM remote_session WHERE remote_session = ? AND tstamp >= UNIX_TIMESTAMP()";
	$session = $app->db->queryOneRecord($sql, $session_id);
	if($session['remote_userid'] > 0) {
		return $session;
	} else {
		throw new SoapFault('session_does_not_exist', 'The Session is expired or does not exist.');
		return false;
	}
{{< / highlight >}}

Normally, a session ID is a random string of alphanumeric characters, for example `1f9aefbe41ebbc0e1e154d1cccd27718`.
They are valid until the implementation that calls them calls `logout`, or, if they never do, for a default length of
time (at the time of discovery this was 30 minutes).

So, what happens if we send the integer `1`, and the token above is a valid token in the database?

```
# curl -k https://127.0.0.1:8080/remote/json.php?login --data '{"username":"admin","password":"hunter2"}'
{"code":"ok","message":"","response":"1f9aefbe41ebbc0e1e154d1cccd27718"}

# curl -k https://127.0.0.1:8080/remote/json.php?server_get_all --data '{"session_id":1}'
{"code":"ok","message":"","response":[{"server_id":"1","server_name":"ispconfig"}]}
```

Note that we have no quotes around the `1` above - with a real session ID we would, as it's supposed
to be a string, but we force an integer to be used here, and this is inserted into the query as-is.
MySQL then interprets the string it finds in the database (the real session token!) as a number, `1`,
and then compares it to our provided `1` - and they are equal, so we are authenticated!

This allows us to 'hitch a ride' on an existing session, as long as it has already been created, 
the legitimate session ID begins with a digit, and there aren't a lot of leading digits. The number of
requests you need to make increases with the length of the number, for instance if you brute force starting
at 1, and the ID is `99999be41ebbc0e1e154d1cccd27718`, you are going to have to make 99,999 requests.

In practice though, it is very common that the ID's end up something useful like `14e...`. Additionally,
many implementations of the client library I found were using a new session for every single interaction
with the server, generating lots of ID's frequently, and even more usefully, some of them *never* called
`logout`, so these sessions were available for the entire 30 minute window every time.

### Conclusion

I reported this vulnerability to the ISPConfig team on October 18 2021, I had a reply 18 minutes(!) later,
and they let me know of a fix being in development the next morning. Their patch and advisory were issued
on October 21 2021, with version 3.2.7. A very decent turnaround.

The patch contained a fix for the issue, as well as some further hardening of the API in general. The
main fix here was enforcing that the parameters are passed as strings even when provided through the API.

If you are implementing an application and want to avoid this issue, the best way would be for your
application to be aware of the type of data it is receiving, and pass them to the database in a parameterised
fashion, using a library which allows you to tell the database layer what type the data is (or an ORM
which handles this matter for you transparently, if available). However, it is a big effort to retrofit
this onto an existing complex application, so it makes sense for ISPConfig not to go this route,
when they had a vulnerability that needed to be fixed.

After I gained access using this vulnerability, I found a further issue that allowed for escalating
from the API access to a shell. ISPConfig began working on a fix for this issue before I provided
the details (we had spoken about it vaguely, and they identified the cause from context). In any case,
they considered this lower severity as the API access alone is already considered to be admin access, and
that seems sensible to me.
