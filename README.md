Github of http://mailfud.org/postpals/

# Postpals

Postpals is a simple policy daemon for Postfix. It maintains a database of outgoing mail, specifically recipients and relays associated to them.

## Introduction

Postpals is a simple policy daemon for Postfix. It maintains a database of outgoing mail, specifically recipients and relays associated to them. Main goal is to whitelist mail coming back from those senders and relays early in the postfix restriction chain, so it doesn't get hit with any client UCE checks you are using (RBL, helo, PTR, greylisting etc). This is important because many legimate servers are located in dynamic looking networks etc, which commonly result in false rejects.

Preliminary evidence suggests that any relay (or a /24 subnet of) accepting mail is most likely a legimate one. Thus it shouldn't be rejected or delayed when sending mail back. Other good method is to check known sender/recipient email pairs regardless of relay. For a certain large organization, 28% of total traffic matched a known entry and only 0.1% of those were spam. Most of that spam originated from large relays that should not be rejected directly at MTA anyway.

Postpals is written in Perl for easy maintenance and has only few non-standard module dependencies:
- IO::Multiplex (could do without, but why duplicate existing proven code?)
- Net::Server (ditto above, only Net::Server::Daemonize used)
- File::Tail (for postpals-taillog)

Postpals-taillog is a daemon used to continuously tail postfix mail log and to send found sender/recipient/relay entries to the main postpals daemon database. That way the main daemon is more robust and not limited in which way to receive information.

## Installation

No help for now. Just download the scripts to /usr/local/sbin or such. Then you need to create startup scripts and users as necessary, it depends on your system. Have a look at options the scripts provide and examples below.

## Basic example

Let daemon listen 127.0.0.1:10040 as user postpals, responding OK when entry is found and DUNNO otherwise. If you don't use SpamAssassin, then you probably don't need to keep database of exact relay IPs, so just relay24 should be enough.
````
postpals -u postpals -g postpals -d /var/postpals/postpals.db -T relay24,rcpt 10040
````
Configure postfix main.cf to check policy service before other checks. They will be skipped on postpals matching. Remember to put all relay checks first, so you don't create an open relay!
````
smtpd_recipient_restrictions =
  ...
  reject_unauth_destination
  ...
  check_policy_service inet:127.0.0.1:10040
  ...
  (RBL, helo, PTR checks..)
````
Optionally prime an empty database with some old logs from stdin. You can also test that it works properly this way.
````
postpals-taillog -x -v 10040 < /var/log/mail.log
````
Run the tailing daemon (as someone who has permission to read the log):
````
postpals-taillog -u syslog 10040 /var/log/mail.log
````

## Basic example addition (how to add a header)

Due to Postfix design, it gets a bit trickier if you want to PREPEND a header and at the same time OK the request.
Start postpals with -C option, so it responds with DUNNO or restriction class named postpals_hit_<testname>.
````
postpals -C -u postpals -g postpals -d /var/postpals/postpals.db 10040
````
You also need to define the classes in postfix main.cf:
````
smtpd_restriction_classes =
  postpals_hit_relay
  postpals_hit_relay24
  postpals_hit_rcpt

postpals_hit_relay =
  check_sender_access pcre:/etc/postfix/postpals_hit_relay.pcre
  permit_auth_destination

postpals_hit_relay24 =
  check_sender_access pcre:/etc/postfix/postpals_hit_relay24.pcre
  permit_auth_destination

postpals_hit_rcpt =
  check_sender_access pcre:/etc/postfix/postpals_hit_rcpt.pcre
  permit_auth_destination
````
When postpals responds to a match with class name, a dummy pcre table (or regexp if you like) will add a header to the mail. Then rest of the postfix checks are skipped using permit_auth_destination. Example of the pcre table:
````
/^/ PREPEND X-Postpals-Hit: relay
````
You should use an unguessable header name like X-Postpals-83751, so it can't be forged easily!

You can then use your header in SpamAssassin to score and log hits.
````
header POSTPALS_RELAY24 X-Postpals-Hit =~ /\brelay24\b/
score POSTPALS_RELAY24 -0.5
header POSTPALS_RELAY X-Postpals-Hit =~ /\brelay\b/
score POSTPALS_RELAY -1
header POSTPALS_RCPT X-Postpals-Hit =~ /\brcpt\b/
score POSTPALS_RCPT -1.5
````

## Nitty Gritty Details

Postpals is a single process multiplexing daemon. Design goal was to remove need for bloaty external databases like BerkeleyDB. The data is stored in memory as Perl hashes and periodically backed up to disk (using Storable module). All sender/recipient keys are reduced to 5-byte hashes and IPs are stored as integers, so memory usage is smallish (for Perl..) and disk backup takes little space. Tested peak performance ranges from 350 queries/s (P2 350MHz) to 6500 queries/s (AMD 3GHz).

Taking an userbase of 2000 with 20000 msgs/day as an example: database contains 100000 keys using 10MB memory, 1MB disk backup. Cleanup and backup run takes less than a second.

It's possible to feed postpals with your own tool instead of postpals-taillog. Only thing needed is to send a special policy request to postpals socket:
````
request=postpals_add_entry
sender=local.user@your.domain
recipient=remote.user@somewhere.far
relay=1.2.3.4
[empty line]
````
Required info is either relay (for relay/relay24 tests) and/or sender+recipient (for rcpt test). Likewise a postpals_del_entry request will delete keys from the database. For performance reasons the daemon will not reply back.

## License

Postpals is free software and released under BSD license, which basically means that you can do what you want as long as you keep the copyright notice:

Copyright © 2010, Henrik Krohns <postpals@hege.li>
All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

- Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.
- Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.
- Neither the name of the copyright holders nor the names of its contributors may be used to endorse or promote products derived from this software without specific prior written permission.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.
