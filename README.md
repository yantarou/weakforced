Weakforced
----------
The goal of 'wforced' is to detect brute forcing of passwords across many servers,
services and instances. In order to support the real world, brute force detection
policy can be tailored to deal with "bulk, but legitimate" users of your service,
as well as botnet-wide slowscans of passwords.

Here is how it works:
 * Report successful logins via JSON http-api
 * Report unsuccessful logins via JSON http-api
 * Query if a login should be allowed to proceed, should be delayed, or ignored via http-api

Runtime console for querying the status of logins, IP addresses, subnets.

wforced is aimed to receive message from services like:

 * IMAP
 * POP3
 * Webmail logins
 * FTP logins
 * Authenticated SMTP
 * Self-service logins
 * Password recovery services

By gathering failed and successful login attempts from as many services as
possible, brute forcing attacks can be detected and prevented more
effectively. 

Inspiration:
http://www.techspot.com/news/58199-developer-reported-icloud-brute-force-password-hack-to-apple-nearly-six-month-ago.html

Policies
--------

There is a sensible default policy, and extensive support for crafting your own policies using
the insanely great Lua scripting language. 

Sample:

```
function allow(wfdb, lt)
	if(wfdb:countDiffFailuresAddress(lt.remote, 10) > 50)
	then
		return -1 -- BLOCK!
	end

	if(wfdb:countDiffFailuresAddressLogin(lt.remote, lt.login, 10) > 3)
	then
		return 3  -- must wait for 3 seconds 
	end

	return 0          -- OK!
end
```

Many more metrics are available to base decisions on. Some example code is in [wforce.conf](wforce.conf).

To report (if you configured with 'webserver("127.0.0.1:8084", "secret")'):

```
$ for a in {1..101}
  do 
    curl -X POST --data '{"login":"ahu", "remote": "127.0.0.1", "pwhash":"1234'$a'", "success":"false"}' \
    http://127.0.0.1:8084/?command=report -u wforce:secret
  done 
```

This reports 101 failed logins for one user, but with different password hashes.

Now to look up if we're still allowed in:

```
$ curl -X POST --data '{"login":"ahu", "remote": "127.0.0.1", "pwhash":"1234"}' \
  http://127.0.0.1:8084/?command=allow -u ahu:super
{"status": -1}
```

It appears we are not!

Spec
----
We report 4 fields in the LoginTuple

 * login (string): the user name or number or whatever
 * remote (ip address, no power): the address the user arrived on
 * pwhash (string): a highly truncated hash of the password used
 * succes (boolean): was the login a success or not?

All are rather clear, but pwhash deserves some clarification. In order to
distinguish actual brute forcing of a password, and repeated incorrect but
identical login attempts, we need some marker that tells us if passwords are
different.

Naively, we could hash the password, but this would spread knowledge of
secret credentials beyond where it should reasonably be. Even if we salt and
iterate the hash, or use a specific 'slow' hash, we're still spreading
knowledge.

However, if we take any kind of hash and truncate it severely, for example
to 12 bits, the hash tells us very little about the password itself - since
one in 4096 random strings will match it anyhow. But for detecting multiple
identical logins, it is good enough.

For additional security, hash the login name together with the password - this
prevents detecting different logins that might have the same password.

NOTE: wforced does not require any specific kind of hashing scheme, but it
is important that all services reporting successful/failed logins use the
same scheme!

When in doubt, try:

```
TRUNCATE(SHA256(LOGIN + '\x00' + PASSWORD), 12)
```

Which denotes to take the first 12 bits of the hash of the concatenation of
the login, a 0 bytes and the password. Prepend 4 0 bits to get something
that can be expressed as two bytes.

API Calls
---------
We can call 'report', 'allow' and (near future) 'clear', which removes
entries from a listed 'login' and/or 'remote'. 

Load balancing: siblings
------------------------
For high-availability or performance reasons it may be desireable to run
multiple instances of wforced. To present a unified view of status however,
these instances then need to share the login tuples. To do so, wforce
implements a simple knowledge-sharing system.

Tuples received are broadcast (best effort, UDP) to all siblings. The
sibling list is parsed such that we don't broadcast messages to ourselves
accidentally, and can thus be identical across all servers.

To define siblings, use:

```
addSibling("192.168.1.79")
addSibling("192.168.1.30")
addSibling("192.168.1.54")
siblingListener("0.0.0.0")
```

This last line configures that we also listen to our other siblings (which
is nice).  The default port is 4001, the protocol is UDP.

With this setup, several wforces are all kept in sync, and can be load
balanced behind for example haproxy, which incidentally can also offer SSL.


