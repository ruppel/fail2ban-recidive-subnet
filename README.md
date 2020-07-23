# fail2ban-recidive-subnet

Find and ban recidive subnets using fail2ban.

## Problem

[Fail2ban](https://github.com/fail2ban/fail2ban) is perfect to ban single hosts that cause authentication errors or try some other bad stuff on your server.
But watching the fail2ban.log I discovered that some attackers use ip ranges.
Some ip's are found by fail2ban but not banned, because the attack is distributed from a complete subnet.
Fail2ban by now is not able to discover and ban those subnets.
There is an issue for that [#927](https://github.com/fail2ban/fail2ban/issues/927) and a python solution [fail2ban-subnet](https://github.com/XaF/fail2ban-subnets).
But both did't satisfy my needs, so this little project is born.

## How it works

There is a shell script, that starts an awk script which scans the fail2ban.log file.
The shell script should be started on a regular basis (e.g. cronjob).
Every ip where fail2ban said "Found" is remembered by the awk script.
If the /24 subnet of that ip has too many "Founds", then the subnet is written in the fail2ban-subnet.log file.
Now fail2ban itself has a jail and filter configuration to watch this log file and will ban and unban the subnet according to your configuration.

## Prerequisites

- fail2ban installed and working (tested with v0.10.2)
- gawk installed

## Installation

- Copy the scripts `fail2ban-subnet.awk` and `fail2ban-subnet-starter.sh` to a location of your choice
- Ensure the `fail2ban-subnet-starter.sh` is executable (`chmod +x ...`)
- Copy `iptables-subnet.local` to `/etc/fail2ban/action.d/`
- Copy `recidive-subnet.local` to `/etc/fail2ban/filter.d`
- Add filter definition in `jail.local` to your `/etc/fail2ban/jail.local` (at the very end)
- Configure settings in `fail2ban-subnet-starter.sh`
- Configure settings in `/etc/fail2ban/jail.local` (esp. bantime)
- Restart fail2ban (depends on your unix distro)
- Call your `fail2ban-subnet-starter.sh` (for the first and the last time by hand)
- Add `fail2ban-subnet-starter.sh` to your cron to be started regularly (e.g. every hour)

## Configuration

### `fail2ban-subnet-starter.sh`

- `findtime`
  - Time period in seconds
  - A subnet is banned if it generated more than `maxips` "Found" messages with different ip's in the last `findtime` seconds
  - My setting: 5 days
    - `findtime=$((5*24*60*60))`
- `excluded_jails`
  - String with jailnames separated with \<space>
  - The jailnames given in `excluded_jails` will not be checked
  - My setting: exclude recidive and recidive-subnet (myself)
    - `excluded_jails="recidive recidive-subnet"`
- `maxips`
  - `maxips` is the number of maximum different ips from a /24 subnet with a "Found" in the fail2ban logfile before the subnet gets banned.
  - smaller numbers make the tool more "aggressive".
  - In my opinion this should be not smaller than 3
  - My setting:
    - `maxips=5`
- `logfilename`
  - The path and the name of the fail2ban logfile
  - My setting:
    - `logfilename=/var/log/fail2ban.log`
- `outputfile`
  - File path and name where to log the found subnet
  - Should correspond to the parameter `logpath` in the `[recidive-subnet]` jail definition in `jail.local`
  - My setting:
    - `outputfile=/var/log/fail2ban-subnet.log`
- `awkscript`
  - File path and name where you stored the awk script `fail2ban-subnet.awk`
  - My setting:
    - `awkscript=/opt/fail2ban-subnet/fail2ban-subnet.awk`

### `jail.local`

Fail2ban recommends to not directly change the `jail.conf` file, but to create a `jail.local` file with your local configuration. So if you correctly installed and configured fail2ban you should have a `jail.local` file.

The `jail.local` file given in this repository should not override your `jail.local` file but you should copy the content at the end of your existing `jail.local`.

Then you can adapt the configuration to your needs.

- `enabled`
  - This enables or disables the jail
  - My setting:
    - `enabled = true`
- `action`
  - The action to start, if fail2ban found a recidive subnet
  - This is a copy of the default action `action_mwl` which bans and sends a mail with whois information, but I changed the `%(banaction)s` to `iptables-subnet` which is my own defined action for banning /24 subnets
  - My setting:
    - `action = iptables-subnet[name=%(__name__)s, port="%(port)s", protocol="%(protocol)s", chain="%(chain)s"`
    - `%(mta)s-whois-lines[name=%(__name__)s, dest="%(destemail)s", logpath=%(logpath)s, chain="%(chain)s"]`
- `logpath`
  - The path to the file which the filter should parse
  - This is the file, which is written by the scripts above
  - My setting:
    - `logpath = /var/log/fail2ban-subnet.log`
- `bantime`
  - Time period how long the subnet should be banned
  - As I'm really bored by those attackers, I give them half a year!
  - My setting:
    - `bantime = 26week`
- `findtime`
  - A subnet is banned if it has generated `maxretry` logs during the last `findtime` seconds.
  - A change only makes sense, if you change the `maxretry` to something other than `1`
  - My setting:
    - `findtime = 1day`
- `maxretry`
  - `maxretry` is the number of founds before a subnet get banned.
  - As most of the work is done by the scripts which generate the `fail2ban-subnet.log` once a subnet is found it should be banned directly, so I set `maxretry` to `1`
  - If you don't want to be as aggressive as I, then better change the `maxips` config above to a higher value or lower the `bantime`
  - My setting:
    - `maxretry = 1`
