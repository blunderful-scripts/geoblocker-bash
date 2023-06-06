# geoblocker_bash
Automatic geoip blocker for Linux based on a whitelist for a specific country.

Suite of bash scripts with easy install and uninstall, focusing on reliability. Fetches, parses and validates an ipv4 whitelist for a given country, then blocks incoming traffic from anywhere except whitelisted subnets. Implements automatic update of the whitelist. Uses iptables.

The ip list is fetched from RIPE - regional Internet registry for Europe, the Middle East and parts of Central Asia. RIPE appears to store ip lists for countries in other regions as well, although I did not check every country.

Intended use case is a server/computer that needs to be publically accessible in your country but does not need to be internationally accessible.

**TL;DR**

Recommended to read the NOTES section below.

To install:
1) Download the latest realease:
https://github.com/blunderful-scripts/geoblocker_bash/releases
2) Install prerequisites. On Debian and derivatives run: sudo apt install ipset jq wget grepcidr
3) Download *all* scripts in this suite into the same folder
4) Use the check_ip_in_ripe script to make sure that your public ip address is included in the list fetched from RIPE, so you do not get locked out of your server.
example: 'bash check_ip_in_ripe.sh -c DE -i <your_public_ip_address>' (for Germany)
5) Once verified that your public ip address is included in the list, run 'sudo bash geoblocker_bash-install -c <country_code>'
example: 'sudo bash geoblocker_bash-install -c DE' (for Germany)
 
 To uninstall:
 run "sudo geoblocker_bash-uninstall"

**Prerequisites**:
- Linux running systemd (tested on Debian and Mint, should work on any Debian derivative, may require modifications to work on other distributions)
- Root access
- iptables (default firewall management utility on most linux distributions)
- standard GNU utilities including awk, sed, grep

additional prerequisites: to install, run 'sudo apt install ipset wget jq grepcidr'
- wget (or alternatively curl) is used by the "fetch" and "check_ip_in_ripe" scripts to download lists from RIPE
- ipset utility is a companion tool to iptables (used by the "apply" script to create an efficient iptables whitelist rule)
- jq - Json processor (used to parse lists downloaded from RIPE)
- grepcidr - filters ip addresses matching CIDR patterns (used by check_ip_in_ripe.sh to check if an ip address belongs to a subnet from a list of subnets)

**Detailed description**

The suite includes 7 scripts:
1. geoblocker_bash-install
2. geoblocker_bash-uninstall
3. geoblocker_bash-run
4. geoblocker_bash-fetch
5. geoblocker_bash-apply
6. validate_cron_schedule.sh
7. check_ip_in_ripe.sh

**The install script**
- Checks prerequisites
- Creates system folder to store data in /var/lib/geoblocker_bash.
- Copies all scripts included in this suite to /usr/local/bin
- Creates backup of pre-install policies for INPUT and FORWARD chains
- Calls geoblocker_bash-run to immediately fetch and apply new firewall config.
- Verifies that crond service is enabled. Enables it if not.
- Validates optionally user-specified cron schedule expression (default schedule is "0 4 * * *" - at 4:00 [am] every day).
- Creates periodic cron job based on that schedule and a reboot job. Cron jobs implement persistence and automatic list updates.
- If an error occurs during installation, calls the uninstall script to revert any changes made to the system.

**The uninstall script**
- Deletes associated cron jobs
- Restores pre-install state of default policies for INPUT and FORWARD chains
- Deletes associated iptables rules and removes the whitelist ipset
- Deletes scripts' data folder /var/lib/geoblocker_bash
- Deletes the scripts from /usr/local/bin

**The run script** simply calls the fetch script, then calls the apply script, passing required arguments. Used for easier triggering from cron jobs.

**The fetch script**
- Fetches ipv4 subnets for a given country from RIPE
- Parses, validates and compiles the downloaded (JSON formatted) list into a plain list, and saves that to a file
- Attempts to determine the local ipv4 subnet for the main network interface and appends it to the file

**The apply script**
- Creates or updates an ipset (essentially an efficiency-optimized ip list) from a whitelist file
- Creates iptables rule that allows connection from subnets included in the ipset
- Sets default policy on INPUT and FORWARD iptables chains to DROP (effectively blocking all connections except what's explicitly allowed by the rules)
- Once all above completed successfully, saves a backup of the current (known-good) iptables state and the current ipset
- In case of an error, attempts to restore last known-good state from backup
- If that fails, the script assumes that something is broken and calls the uninstall script which will attempt to remove any rules we have set, delete the associated cron jobs and restore policies for INPUT and FORWARD chains to the pre-install state

**The validate_cron_schedule.sh script** is used by the install script. It accepts cron schedule expression and attempts to make sure that it complies with the format that cron expects. Used to validate optionally user-specified cron schedule expressions.

**The check_ip_in_ripe.sh script** can be used to verify that a certain ip address belongs to a subnet found in RIPE's records for a given country.

**NOTES**

1) I have put much effort into ensuring reliable operation of the scripts and implementing fallback and recovery mechanisms in case of an error. Yet, I can not guarantee that they will work as intended (or at all) in your environment. You should evaluate and test by yourself.

2) Changes applied to iptables are made persistent via cron jobs: a periodic job running at a daily schedule (which you can optionally change when running the install script), and a job that runs at system reboot (after 30 seconds delay).

3) You can specify a custom schedule for the periodic cron job by passing an argument to the install script. Run it with the '-h' switch for more info.

4) **Note** that cron jobs **will be run as root**.

5) To test before deployment, you can run the install script with the "-n" switch to skip creating cron jobs. This way, a simple server restart will undo all changes made to the firewall. To enable persistence later, install again without the "-n" switch.

6) The run, fetch and apply scripts write to syslog in case an error occurs. The run script also writes to syslog upon success. To verify that cron jobs ran successfully, on Debian and derivatives run 'sudo cat /var/log/syslog | grep geoblocker_bash'

7) I would love to hear whether it works or doesn't work on your system (please specify which), or if you find a bug, or would like to suggest code improvements. You can use the "Discussions" or "Issues" tab for that.
