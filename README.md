## Setting Geoip capabilities with ufw/iptables  with automatic IP addr db updates on CloudPanel Hosting.

1.  `sudo apt install curl perl unzip xtables-addons-common libtext-csv-xs-perl libmoosex-types-netaddr-ip-perl`
2. Prepare GeoIP database manually

```
sudo mkdir /usr/share/xt_geoip
cd /usr/share/xt_geoip
sudo /usr/libexec/xtables-addons/xt_geoip_dl
sudo /usr/libexec/xtables-addons/xt_geoip_build -s -i dbip*.csv
```
4. Put this into  `/etc/cron.weekly/geoip-update`

```
#!/bin/bash -e

WORKDIR=`mktemp -d`
if [[ ! "$WORKDIR" || ! -d "$WORKDIR" ]]; then
        echo "Error: Could not create temp dir on $(date)" >> /var/log/geoip_update.log
        exit 1
fi

cd $WORKDIR
/usr/libexec/xtables-addons/xt_geoip_dl
/usr/libexec/xtables-addons/xt_geoip_build -q -s -i dbip*.csv

# Clean up WORKDIR
cd /tmp
rm -rf $WORKDIR

# Reload UFW so rules using geoip match new database
ufw reload

# Log the cleanup activity
echo "Weekly geoip update completed on $(date)" >> /var/log/geoip_update.log
```
Then,

`chmod +x /etc/cron.weekly/geoip-update`

Test `/etc/cron.weekly/` with `run-parts`

`run-parts --test /etc/cron.weekly/`

This line should appear on the output

`/etc/cron.weekly//geoip-update`

3.  `modprobe xt_geoip`
and check

`lsmod | grep ^xt_geoip`

The output should be similar to this.
```
xt_geoip             16384  34
```
    
5.  Add rules to UFW file "`/etc/ufw/before.rules`". Prepend before  `COMMIT`  (at the very end).
    
```
#### CUSTOM EXAMPLE:
# block all traffic from RU,CN
-A ufw-before-input -p tcp -m geoip --src-cc RU,CN -j DROP

# block all traffic to port 22 from all ips except TH
-A ufw-before-input -p tcp --dport 22 -m geoip ! --src-cc TH -j DROP

# Allow TH
-A ufw-before-input -m geoip --src-cc TH -j ACCEPT

# Log non-TH traffic
-A ufw-before-input -m geoip ! --src-cc TH -j LOG --log-prefix "[UFW BLOCKED COUNTRIES] "

# Drop non-TH traffic
-A ufw-before-input -m geoip ! --src-cc TH -j DROP
#### END CUSTOM
```
6. Reload UFW
    `ufw reload`

## Automated country stats script

**Use `geoiplookup` (simpler)**

Ubuntu has the `geoip-bin` package which gives `geoiplookup`:

`sudo apt install geoip-bin` 

Then test:

`geoiplookup 8.8.8.8` 

Output:

`GeoIP Country Edition:  US,  United  States` 

Get only country code:

`country=$(geoiplookup "$ip" | awk -F: '{print $2}' | awk -F, '{print $1}' | sed "s/^ *//;s/ *$//")`

Create `/usr/local/bin/geoip-drop-stats.sh`:
```
#!/bin/bash
LOGFILE="/var/log/ufw.log"
TMPFILE="/tmp/geoip-ips.txt"

# Extract dropped IPs
grep "UFW BLOCKED COUNTRIES" "$LOGFILE" | awk '{for(i=1;i<=NF;i++){if($i ~ /^SRC=/){print substr($i,5)}}}' | sort -u > "$TMPFILE"

echo "===== Blocked connections by country ====="
while read -r ip; do
    country=$(geoiplookup "$ip" | awk -F: '{print $2}' | awk -F, '{print $1}' | sed "s/^ *//;s/ *$//")
    echo "$country"
done < "$TMPFILE" | sort | uniq -c | sort -nr
```

Make executable:

`sudo chmod +x /usr/local/bin/geoip-drop-stats.sh` 

Run:

`sudo /usr/local/bin/geoip-drop-stats.sh` 

Example output:
```
===== Blocked connections by country =====
    145 US
     19 VN
     14 CN
     14 AU
      9 SG
      9 AR
      6 GB
```

## CloudPanel UFW Tweak

After Add/Edit and Save firewall rules in Security/Firewall menu on Admin Area, CloudPanel will reset `ufw before.rules` and afrer.rules to default.

The default rules of ufw are located in `/usr/share/ufw/iptables`.

Firstly backup the old rules.

```
cd /usr/share/ufw/iptables
mv before.rules before.rules.ORIG
```
 
Then copy our before.rules 

```
cd /usr/share/ufw/iptables
cp /etc/ufw/before.rules .
```

OK, now you can Add/Edit and Save firewall rules in Security/Firewall menu on CloudPanel without effect on `before.rules` with xtables geoip.
