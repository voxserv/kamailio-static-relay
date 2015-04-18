Static SIP and RTP relay proxy
==============================

Introduction
------------

This project sponsored by BluePackets - http://www.bluepackets.com.au

This is a Kamailio configuration that builds up a static SIP and RTP
proxy and relays the packets between two IP interfaces on the server and
two remote SIP servers. It allows to hide the internal network topology
and go around some security or topology restrictions.

The network topology consists of two segments: internal and
external. The relay server has IP interfaces in both segments, and
forwards SIP messages and RTP data in both directions.

The following IP addresses and ports are used further in this document:

Internal SIP server: `10.0.0.10:5060`
Relay server's interface in internal network: `10.5.5.5:5060`
Relay server's interface in external network: `192.0.2.5:5060`
External SIP server: `198.51.100.100:5060`


When `10.0.0.10` needs to send a SIP call to `198.51.100.100`, it sends
it to `10.5.5.5`, and when `198.51.100.100` needs to send a call to
`10.0.0.10`, it sends it to `192.0.2.5`. Both SIP servers see the
corresponding relay's interface IP address as the source address in
received packets. The RTP media is relayed using the same principle.

The configuration is completely symmetrical, but the internal and
external addresses should not be mixed up.


Software installation
---------------------

The following is the installation sequence for Debian Wheezy.

```
curl http://deb.kamailio.org/kamailiodebkey.gpg | apt-key add -

cat >/etc/apt/sources.list.d/kamailio.list <<EOT
deb http://deb.kamailio.org/kamailio wheezy main
EOT

apt-get update && apt-get install -y kamailio rtpproxy git

cat >/etc/default/kamailio <<'EOT'
# Kamailio startup options
RUN_KAMAILIO=yes
USER=kamailio
GROUP=kamailio
SHM_MEMORY=64
PKG_MEMORY=8
CFGFILE=/etc/kamailio/kamailio.cfg
DUMP_CORE=yes
EOT

cat >/etc/rsyslog.d/siprelay.conf <<'EOT'
local0.*                        -/var/log/kamailio.log
local1.*                        -/var/log/rtpproxy.log
EOT

service rsyslog restart 

cat >/etc/logrotate.d/siprelay <<'EOT'
/var/log/kamailio.log
/var/log/rtpproxy.log
{
        rotate 4
        weekly
        missingok
        notifempty
        compress
        delaycompress
        sharedscripts
        postrotate
                invoke-rc.d rsyslog rotate > /dev/null
        endscript
}
EOT
```

`sngrep` is a convenient tool for debugging SIP sessions. It requires a
wide text terminal with color support (xterm, for example). It is not
required, but very handy in troubleshooting and tuning a SIP service:

```
apt-get update && \
apt-get install -y git autoconf automake gcc make libncurses5-dev \
libpcap-dev libssl-dev libpcre3-dev
cd /usr/local/src
git clone https://github.com/irontec/sngrep.git
cd sngrep/
./bootstrap.sh && ./configure && make install
```


Proxy configuration
-------------------

All site-specific configuration is done in `kamailio_siteconfig.cfg`,
and this file is ignored by Git. This allows you to keep the Github
remote for future updates. You can also create your own branch where
`kamailio_siteconfig.cfg` is excluded from `.gitignore`, and merge the
updates from Github with your branch.

```
cd /etc/kamailio
/bin/rm *
git clone https://github.com/voxserv/kamailio-static-relay.git .

cat >kamailio_siteconfig.cfg <<'EOT'
#!substdef "/INTERNAL_HOST/10.0.0.10/"
#!substdef "/INTERNAL_PORT/5060/"
#!substdef "/EXTERNAL_HOST/198.51.100.100/"
#!substdef "/EXTERNAL_PORT/5060/"
port=5060
EOT


# -l option specifies the relay NIC addresses, and the order is important:
# internal NIC address should go first. The UDP port range is up to you,
# and it should be consistent with all surrounding firewall configurations

cat >/etc/default/rtpproxy <<'EOT'
CONTROL_SOCK=udp:127.0.0.1:9000
EXTRA_OPTS="-l 10.5.5.5/192.0.2.5 -m 10000 -M 40000 -d WARN:LOG_LOCAL1"
EOT

insserv rtpproxy
service rtpproxy restart

insserv kamailio
service kamailio start
```


Author
------

Copyright (c) 2015 Stanislav Sinyagin <ssinyagin@k-open.com>
http://voxserv.ch/


Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in
all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
THE SOFTWARE.