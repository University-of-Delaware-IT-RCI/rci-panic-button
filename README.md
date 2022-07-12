# rci-panic-button

When your data center contains large, hot HPC clusters and finicky HVAC systems, risk management dictates that staff are able to quickly shutdown the majority of heat-generating components of the clusters.  Thus, `rci-panic-button` was written.

Checking the power status of all nodes or powering them down via IPMI is a straighforward processing of the list of hostnames.  The utility checks whether or not the IPMI interface can be reached via ping, then sends the IPMI command.  The Python multiprocessing module is used to parallelize the action across the list of hostnames, and results are printed as they are yielded.

Powering all nodes on is a slightly more difficult procedure.  Our clusters use the PERCEUS/Warewulf provisioning system, and some of the VNFS images are rather large.  Booting all 200+ nodes at the same time, some of the nodes will inevitably end up waiting for the VNFS image and probably timing-out before they are able to download it.  On older clusters we also found that dhcpd had trouble servicing all of the overlapping DHCP requests in a timely fashion.  To mitigate these issues, the utility attempts to boot *N* nodes at a time:

- active list = empty list
- While there are hosts left to boot:
    - Note timestamp, t0
    - Does the active list NOT contain *N* hosts?
        - *(N - len(active list))* nodes are powered-on
        - Any nodes that fail ping or IPMI are unreachable, summarize and DO NOT add to active list...
        - ...otherwise the key-value pair hostname : timestamp is added to the active list
    - Does the active list contain any hosts?
        - Check each host for connectivity to sshd on port 22
        - Any nodes that connect okay, summarize and REMOVE from active list
        - Any nodes that do not connect and have a timestamp value (in the active list) indicating more than 10 minutes have elapsed, summarize and REMOVE from active list
        - Does the active list contain any hosts?
            - Note timestamp, t1
            - If (t1 - t0) is less than a minimum amount of time, sleep until that much time has passed

The minimum amount of time per iteration is present to avoid putting the utility into a spin loop that hammers on node ssh ports and powers-on more nodes too quickly.  If the active list ends up being empty at the end of the iteration, it implies that every in-flight node has met booted-or-unreachable criteria and there's no reason to throttle-back the next set of power-on events.

## Usage

The utility should work with any version of Python >= 2.6 and has built-in help:

```
$ sudo /usr/local/sbin/rci-panic-button --help
usage: rci-panic-button [-h] [--nprocs #] [--yes] [--bootup-stride #]
                        [--bootup-max-wait <seconds>]
                        [--bootup-min-check <seconds>]
                        [{status,down,up,off,on}]

Attempt to manage power of all compute nodes.

positional arguments:
  {status,down,up,off,on}
                        Operation to perform (default: status)

optional arguments:
  -h, --help            show this help message and exit
  --nprocs #, -N #      Maximum number of parallel processes to use
  --yes, -y             Initial confirmation that you do indeed want to affect
                        the compute nodes
  --bootup-stride #, -s #
                        Number of nodes to power-on concurrently (0 implies
                        all at once, default 15)
  --bootup-max-wait <seconds>, -w <seconds>
                        Maximum amount of time to wait for a powered-on node
                        to answer to ssh (default 600 seconds)
  --bootup-min-check <seconds>, -m <seconds>
                        Minimum amount of time each bootup iteration should
                        take (limits how fast status is rechecked and new
                        hosts started, default 30 seconds)
```

Default values for each `--bootup-*` option and the `--nprocs` option should be hard-coded into the `Config` class in the utility.  Additional modifications — like IPMI hostname, username, and password generator functions — should be altered when adapting the utility for a specific cluster.

In order for any of the actions to be performed, the user must provide the `--yes` or `-y` flag.  If it is not provided, the utility will proceed no further than parsing the command line arguments.

### Status mode

The default operational mode, "status," simply displays the IPMI power state for each node:

```
$ sudo /usr/local/sbin/rci-panic-button --yes
INFO:     running in STATUS mode
INFO:     hostlist generator command returned 272 hosts
r00n05                SUCCESS (Chassis Power is on)
r00n10                SUCCESS (Chassis Power is on)
r00n04                SUCCESS (Chassis Power is on)
   :
r00n08                SUCCESS (Chassis Power is on)
r04n57                SUCCESS (Chassis Power is on)
r02s00                SUCCESS (Chassis Power is on)
r04n05                SUCCESS (Chassis Power is on)
r04n09                SUCCESS (Chassis Power is on)
```

Note that the node order will be indeterminate, so the output of this command can be sort'ed if necessary.

### Power-down mode

The "down" or "off" mode is used to shutdown all compute nodes.  The mode must be explicitly provided to the command:

```
$ sudo /usr/local/sbin/rci-panic-button --yes down
INFO:     running in DOWN mode
INFO:  Please type the following phrase verbatim to confirm your intention to
       POWER DOWN all compute nodes in the cluster:

TYPE:  i find your lack of faith disturbing
     > i find your lack fo fayth distrubing
ERROR:    you did not enter the phrase properly, guess you are not serious
```

The last guard against mistakenly shutting down the entire cluster is the verbatim entry of a random phrase (yes, they are all Star Wars quotes).  The phrase displayed must be typed-out correctly in order to proceed:

```
$ sudo /usr/local/sbin/rci-panic-button --yes down
INFO:     running in DOWN mode
INFO:  Please type the following phrase verbatim to confirm your intention to
       POWER DOWN all compute nodes in the cluster:

TYPE:     the ability to speak does not make you intelligent
        > the ability to speak does not make you intelligent
INFO:     congratulations, you typed the phrase properly and indicated you really
          want to affect all the compute nodes...proceeding with that action now
INFO:     hostlist generator command returned 272 hosts
r00g00                SUCCESS (Chassis Power Control: Down/Off)
r00g03                SUCCESS (Chassis Power Control: Down/Off)
r00g02                SUCCESS (Chassis Power Control: Down/Off)
r00n00                SUCCESS (Chassis Power Control: Down/Off)
   :
r04n72                SUCCESS (Chassis Power Control: Down/Off)
r04s00                SUCCESS (Chassis Power Control: Down/Off)
r04n71                SUCCESS (Chassis Power Control: Down/Off)
```

Note that, as expected, the host order was different versus the "status" mode command demonstrated above.

### Power-on mode

The "up" or "on" mode is used to power all compute nodes on.  The mode must be explicitly provided to the command:

```
$ sudo /usr/local/sbin/rci-panic-button --yes up
INFO:     running in UP mode
INFO:  Please type the following phrase verbatim to confirm your intention to
       POWER UP all compute nodes in the cluster:

TYPE:     luminous beings are we not this crude matter
        > luminous beings are we not this crude matter
INFO:     congratulations, you typed the phrase properly and indicated you really
          want to affect all the compute nodes...proceeding with that action now
INFO:     hostlist generator command returned 272 hosts
                    Iteration 1 started with 0 hosts in active state
                    Iteration 1 added 15 hosts to active state
n055                  SUCCESS (powered on, answers to ssh)
test-gpu2             SUCCESS (powered on, answers to ssh)
n057                  SUCCESS (powered on, answers to ssh)
n172                  SUCCESS (powered on, answers to ssh)
                    Iteration 2 started with 11 hosts in active state
                    Iteration 2 added 4 hosts to active state
n048                  SUCCESS (powered on, answers to ssh)
n116                  FAILED (/bin/ping, rc = 1)
n159                  SUCCESS (powered on, answers to ssh)
n064                  SUCCESS (powered on, answers to ssh)
n126                  FAILED (no response to ssh)
  :
```

Note that, as expected, the host order was different versus the other commands demonstrated above.

