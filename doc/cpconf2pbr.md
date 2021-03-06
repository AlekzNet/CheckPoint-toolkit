## cpconf2pbr.py

cpconf2pbr.py creates PBR rules, based on:
* CheckPoint GAIA clish configuration
* List of IP addresses

and adds the IP-addresses from the list to the CP firewall configuration.

### Usage:

```txt
usage: cpconf2pbr.py [-h] [--noclish] [--ifprio IFPRIO] [--rtprio RTPRIO]
                     [--ignore_if IGNORE_IF] [--ignore_ip IGNORE_IP] [--list]
                     [--dst | --src] [--nodbedit] [--table TABLE]
                     [--listprio LISTPRIO] [--fw] [--group GROUP]
                     [conf]

positional arguments:
  conf                  Filename with a list of IP addresses, CheckPoint
                        gateway conf filename, produced by clish -c 'show
                        configuration' or "-" to read from the console
                        (default)

optional arguments:
  -h, --help            show this help message and exit
  --noclish             Do not add "clish -c" construction
  --ifprio IFPRIO       The beginning priority of the PBR rules, related to
                        the interfaces, default=10
  --rtprio RTPRIO       The beginning priority of the PBR rules, related to
                        the local routes, default=100
  --ignore_if IGNORE_IF
                        Comma separated list of interfaces to ignore,
                        default=mgmt,sync,lo
  --ignore_ip IGNORE_IP
                        Comma separated list of IP-addresses to ignore,
                        default=none
  --list                The input is a list of IP-addresses, not a clish
                        config
  --dst                 The list contains the destination addresses
  --src                 The list contains the source addresses
  --nodbedit            Do not add dbedit constructions
  --table TABLE         Table name, default = default
  --listprio LISTPRIO   The beginning priority of the PBR rules for the list
                        of servers, default=1000
  --fw                  Create firewall commands to add the IP-addresses to
                        the config
  --group GROUP         Group name to add the IP-addresses to, default = g-pbr
```

If tested OK, save the configuration with:

```txt
clish -c "save config"
```

### "Exception" PBR rules based on the clish config

This mode is used to create PBR rule to except the local traffic from the PBR. All traffic destined to the directly connected networks or non-default routes will be exempted from the PBR rules with the lower priority.

The interface related table are created with the priority "2", the static route related one with the priority "3".

The source file should be made by the following command:

```txt
clish -c "show configuration"
```

#### Examples

Config lines:

```txt
set interface bond1.2 ipv4-address 1.2.3.1 mask-length 29
set static-route 10.175.255.0/24 nexthop gateway address 163.157.255.129 on
```
Command:

```txt
cat config.txt | cpconf2pbr.py
```

Result:

```txt
clish -c "show configuration" > ~/firewall_clish_before.20190214_0233.conf
clish -c "lock database override "
clish -c "set pbr table tbond1o2 static-route 1.2.3.1/29 nexthop gateway logical bond1.2 priority 2"
clish -c "set pbr rule priority 10 match to 1.2.3.0/29"
clish -c "set pbr rule priority 10 action table tbond1o2"
clish -c "set pbr table t10o175o255o0m24 static-route 10.175.255.0/24 nexthop gateway address 163.157.255.129 priority 3"
clish -c "set pbr rule priority 100 match to 10.175.255.0/24"
clish -c "set pbr rule priority 100 action table t10o175o255o0m24"
clish -c "show configuration" > ~/firewall_clish_after.20190214_0233.conf
```


### PBR rules based on an IP-list

Create the real PBR rules and firewall objects for a list of IP-addresses

#### Examples

List of IP-addresses (note the variety of the supported syntax and absense of sensitivity to intermediate spaces or tabs):

```txt
cat testpbrlist.txt  
1.2.3.4
2.3.4.0/23
 3.4.5.0 255.255.252.0
6.5.4.3/ 255.255.255.0
```


Command to create PBR rules:

```txt
cpconf2pbr.py --list --src  --table deftable testpbrlist.txt
```

Result:

```txt
clish -c "show configuration" > ~/firewall_clish_before.20190214_0233.conf
clish -c "lock database override "
clish -c "set pbr rule priority 1000 match from 1.2.3.4/32"
clish -c "set pbr rule priority 1000 action table deftable"
clish -c "set pbr rule priority 1001 match from 2.3.4.0/23"
clish -c "set pbr rule priority 1001 action table deftable"
clish -c "set pbr rule priority 1002 match from 3.4.5.0/22"
clish -c "set pbr rule priority 1002 action table deftable"
clish -c "set pbr rule priority 1003 match from 6.5.4.3/24"
clish -c "set pbr rule priority 1003 action table deftable"
clish -c "show configuration" > ~/firewall_clish_after.20190214_0233.conf
```

Command to create PBR rules and add the objects to the firewall group g-server-list:

```txt
cpconf2pbr.py --list --src --table deftable --fw --group g-server-list testpbrlist.txt
```

Result:

```txt
################################################################################
# Run these commands on the firewall(s)
################################################################################
clish -c "show configuration" > ~/firewall_clish_before.20190214_0233.conf
clish -c "lock database override "
clish -c "set pbr rule priority 1000 match from 1.2.3.4/32"
clish -c "set pbr rule priority 1000 action table deftable"
clish -c "set pbr rule priority 1001 match from 2.3.4.0/23"
clish -c "set pbr rule priority 1001 action table deftable"
clish -c "set pbr rule priority 1002 match from 3.4.5.0/22"
clish -c "set pbr rule priority 1002 action table deftable"
clish -c "set pbr rule priority 1003 match from 6.5.4.3/24"
clish -c "set pbr rule priority 1003 action table deftable"
clish -c "show configuration" > ~/firewall_clish_after.20190214_0233.conf
################################################################################
# After tested OK, save the config with: clish, save config
################################################################################
################################################################################
# Make a DB backup, then run these commands on the management station
################################################################################
echo -e "create network_object_group g-server-list\nupdate_all\n-q\n" | dbedit -local
echo -e "create host_plain h-001.002.003.004\nupdate_all\n-q\n" | dbedit -local
echo -e "modify network_objects h-001.002.003.004 ipaddr 1.2.3.4\nupdate_all\n-q\n" | dbedit -local
echo -e "addelement network_objects g-server-list '' network_objects:h-001.002.003.004\nupdate_all\n-q\n" | dbedit -local
echo -e "create network n-002.003.004.000_23\nupdate_all\n-q\n" | dbedit -local
echo -e "modify network_objects n-002.003.004.000_23 ipaddr 2.3.4.0\nmodify network_objects n-002.003.004.000_23 netmask 255.255.254.0\nupdate_all\n-q\n" | dbedit -local
echo -e "addelement network_objects g-server-list '' network_objects:n-002.003.004.000_23\nupdate_all\n-q\n" | dbedit -local
echo -e "create network n-003.004.004.000_22\nupdate_all\n-q\n" | dbedit -local
echo -e "modify network_objects n-003.004.004.000_22 ipaddr 3.4.4.0\nmodify network_objects n-003.004.004.000_22 netmask 255.255.252.0\nupdate_all\n-q\n" | dbedit -local
echo -e "addelement network_objects g-server-list '' network_objects:n-003.004.004.000_22\nupdate_all\n-q\n" | dbedit -local
echo -e "create network n-006.005.004.000_24\nupdate_all\n-q\n" | dbedit -local
echo -e "modify network_objects n-006.005.004.000_24 ipaddr 6.5.4.0\nmodify network_objects n-006.005.004.000_24 netmask 255.255.255.0\nupdate_all\n-q\n" | dbedit -local
echo -e "addelement network_objects g-server-list '' network_objects:n-006.005.004.000_24\nupdate_all\n-q\n" | dbedit -local
################################################################################
# Install the new policy
################################################################################
```

Clish and dbedit "decorations" can be removed by `--noclish` and `--nodbedit` correspondingly:

```txt
cpconf2pbr.py --list --src --table deftable --fw --group g-server-list --noclish --nodbedit testpbrlist.txt

set pbr rule priority 1000 match from 1.2.3.4/32
set pbr rule priority 1000 action table deftable
. . .
create network_object_group g-server-list
create host_plain h-001.002.003.004
. . .
```

Make a revision of the database and save config with:
```sh
clish -c "show configuration" > firewall.date.conf
```
in advance.
