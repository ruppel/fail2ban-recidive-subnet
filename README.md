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
