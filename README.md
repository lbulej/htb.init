HTB.init
========

A script to set up HTB traffic control on Linux.


INTRODUCTION
------------

This script is a clone of CBQ.init and is meant to simplify setup of HTB
based traffic control. HTB setup itself is pretty simple compared to CBQ,
3so the purpose of this script is to allow the administrator of large HTB
configurations to manage individual classes using simple, human readable
files.

The "H" in HTB stands for "hierarchical", so while many people did not use
(or know about) the possibility to build hierarchical structures using
CBQ.init, it should be obvious thing to expect from HTB.init :-)

In HTB.init this is done differently, compared to CBQ.init: the usage of
`PARENT` keyword was dropped and instead, class file naming convetion was
introduced. This convention allows the child class to determine ID of its
parent class from the filename and also (if not abused :) enforces file
ordering so that the parent classes are created before their children.

HTB.init uses simple caching mechanism to speed up "start" invocation if the
configuration is unchanged. When invoked for the first time, it compiles the
configuration files into simple shell script containing the sequence of "tc"
commands required to setup the traffic control. This cache-script is stored
in /var/cache/htb.init by default and is invalidated either by presence of
younger class config file, or by invoking HTB.init with "start invalidate".

If you want to HTB.init to setup the traffic control directly without the
cache, invoke it with "start nocache" parameters. Caching is also disabled
if you have logging enabled (ie. `HTB_DEBUG` is not empty).

If you only want HTB.init to translate your configuration to "tc" commands,
invoke it using the "compile" command. Bear in mind that "compile" does not
check if the "tc" commands were successful - this is done (in certain places)
only when invoked with "start nocache" command. When you are testing your
configuration, you should use it to check whether it is completely valid.

In case you are getting strange sed/find errors, try to uncomment line with
`HTB_BASIC` setting, or set the variable to nonempty string. This will enable
function alternatives which require less advanced sed/find functionality. As
a result, the script will run slower but will probably run. Also the caching
will not work as expected and you will have to invalidate the cache manually
by invoking HTB.init with "start invalidate".


CONFIGURATION
-------------

Every traffic class is described by a single file in placed in `$HTB_PATH`
directory, `/etc/sysconfig/htb` by default. The naming convention is different
compared to CBQ.init. The first notable change is a missing 'htb-' prefix. This
was replaced by interface name to improve human readability and to separate
qdisc-only configuration.

Global qdisc options are placed in `$HTB_PATH/<ifname>`, where `<ifname>` is
(surprisingly) name of the interface, made of characters and numbers. This
file must be present if you want to setup HTB on that interface. If you
don't have any options to put into it, leave it empty, but present.

Class options belong to files with names matching this expression:
`$HTB_PATH/<ifname>-<clsid>(:<clsid>)*<description>`

`<clsid>` is class ID which is hexadecimal number in range 0x2-0xFFFF, without
the "0x" prefix. If a colon-delimited list of class IDs is specified, the
last `<clsid>` in the list represents ID of the class in the config file.

`<clsid>` preceding the last `<clsid>` is class ID of the parent class. To keep
ordering so that parent classes are always created before their children, it
is recommended to include full `<clsid>` path from root class to the leaf one.

`<description>` is (almost) arbitrary string where you can put symbolic
class names for better readability.

Examples of valid names:

```
	eth0-2		root class with ID 2, on device eth0
	eth0-2:3	child class with ID 3 and parent 2, on device eth0
	eth0-2:3:4	child class with ID 4 and parent 3, on device eth0
	eth1-2.root	root class with ID 2, on device eth1
```

The configuration files may contain the following parameters. For detailed
description of HTB parameters see http://luxik.cdi.cz/~devik/qos/htb.


### HTB qdisc parameters

The following parameters apply to HTB root queuening discipline only and
are expected to be put into `$HTB_PATH/<ifname>` files. These files *must*
exist (even empty) if you want to configure HTB on given interface.

`DEFAULT=<clsid>`				optional, default 0
`DEFAULT=30`

	<dclsid> is ID of the default class where UNCLASSIFIED traffic goes.
	Unlike HTB qdisc, HTB.init uses 0 as default class ID, which is
	internal FIFO queue that will pass packets along at FULL speed!

	If you want to avoid surprises, always define default class and
	allocate minimal portion of bandwidth to it.

`R2Q=<number>`					optional, default 10
`R2Q=100`

	This allows you to set coefficient for computing DRR (Deficit
	Round Robin) quanta. The default value of 10 is good for rates
	from 5-500kbps and should be increased for higher rates.

`DCACHE=yes|no`					optional, default "no"

	This parameters turns on "dequeue cache" which results in degraded
	fairness but allows HTB to be used on very fast network devices.
	This is turned off by default.


### HTB class parameters

The following are parameters for HTB classes and are expected
to be put into `$HTB_PATH/<ifname>-<clsid>(:<clsid>)*.*` files.

`RATE=<speed>|prate|pceil`			mandatory
`RATE=5Mbit`

	Bandwidth allocated to the class. Traffic going through the class is
	shaped to conform to specified rate. You can use Kbit, Mbit or bps,
	Kbps and Mbps as suffices. If you don't specify any unit, bits/sec
	are used. Also note that "bps" means "bytes per second", not bits.

	The "prate" or "pceil" values will resolve to `RATE` or `CEIL` of
	parent class. This feature is meant to help humans to keep
	configuration files consistent.

`CEIL=<speed>|prate|pceil`			optional, default `$RATE`
`CEIL=6MBit`

	The maximum bandwidth that can be used by the class. The difference
	between `CEIL` and `RATE` amounts to bandwidth the class can borrow,
	if there is unused bandwidth left.

	By default, `CEIL` is equal to `RATE` so the class cannot borrow bandwidth
	from its parent. If you want the class to borrow unused bandwidth, you
	*must* specify the maximal amount it can use, if available.

	When several classes compete for the unused bandwidth, each of the
	classes is given share proportional to their `RATE`.

`BURST=<bytes>`					optional, default computed
`BURST=10Kb`

`CBURST=<bytes>`				optional, default computed
`CBURST=2Kb`

	The `BURST` and `CBURST` parameters control the amount of data that
	can be sent from one class at maximum (hardware) speed before trying
	to service other class.

	If `CBURST` is small (one packet size) it shapes bursts not to
	exceed `CEIL` rate the same way `PEAK` works for TBF.

`PRIO=<number>`					optional, default 0
`PRIO=5`

	Priority of class traffic. The lower the number, the higher
	the priority. Classes with higher priority are offered excess
	bandwidth first.

`LEAF=none|sfq|pfifo|bfifo`			optional, default "none"

	Tells the script to attach the specified leaf queueing discipline
	to HTB class. By default, no leaf qdisc is used.

	If you want to ensure (approximately) fair sharing of bandwidth
	among several hosts in the same class, you should specify `LEAF=sfq`
	to attach SFQ as leaf queueing discipline to the class.

`MTU=<bytes>`					optional, default 1600

	Maximum packet size HTB creates rate maps for. The default should
	be sufficient for most cases, it certainly is for Ethernet.


### SFQ qdisc parameters

The SFQ queueing discipline is a cheap way to fairly share class bandwidth
among several hosts. The fairness is approximate because it is stochastic,
but is not CPU intensive and will do the job in most cases. If you desire
real fairness, you should probably use WRR (weighted round robin) or WFQ
queueing disciplines. Note that SFQ does not do any traffic shaping - the
shaping is done by the HTB class the SFQ is attached to.

`QUANTUM=<bytes>`				optional, qdisc default

	Amount of data in bytes a stream is allowed to dequeue before next
	queue gets a turn. Defaults to one MTU-sized packet. Do *not* set
	this parameter below the MTU!

`PERTURB=<seconds>`				optional, default 10

	Period of hash function perturbation. If unset, hash reconfiguration
	will never take place which is what you probably don't want. The
	default value of 10 seconds is probably a good value.


### PFIFO/BFIFO qdisc parameters

Those are simple FIFO queueing disciplines. They only have one parameter
which determines their length in bytes or packets.

`LIMIT=<packets>|<bytes>`			optional, qdisc default
`LIMIT=1000`

	Number of packets/bytes the queue can hold. The unit depends on
	the type of queue used.


### Filtering parameters

`RULE=[[saddr[/prefix]][:port[/mask]],][daddr[/prefix]][:port[/mask]]`

	These parameters make up "u32" filter rules that select traffic for
	each of the classes. You can use multiple `RULE` fields per config.

	The optional port mask should only be used by advanced users who
	understand how the u32 filter works.

Some examples:

	`RULE=10.1.1.0/24:80`
		selects traffic going to port 80 in network 10.1.1.0

	`RULE=10.2.2.5`
		selects traffic going to any port on single host 10.2.2.5

	`RULE=10.2.2.5:20/0xfffe`
		selects traffic going to ports 20 and 21 on host 10.2.2.5

	`RULE=:25,10.2.2.128/26:5000`
		selects traffic going from anywhere on port 50 to
		port 5000 in network 10.2.2.128

	`RULE=10.5.5.5:80,`
		selects traffic going from port 80 of single host 10.5.5.5



`TOS=<value>`

	These parameters complement the "u32" filter rules, allowing to
	match on the type-of-service field in the IP header.



`REALM=[srealm,][drealm]`

	These parameters make up "route" filter rules that classify traffic
	according to packet source/destination realms. For information about
	realms, see Alexey Kuznetsov's IP Command Reference. This script
	does not define any realms, it justs builds "tc filter" commands
	for you if you need to classify traffic this way.

	Realm is either a decimal number or a string referencing entry in
	/etc/iproute2/rt_realms (usually).

Some examples:

	`REALM=russia,internet`
		selects traffic going from realm "russia" to realm "internet"

	`REALM=freenet,`
		selects traffic going from realm "freenet"

	`REALM=10`
		selects traffic going to realm 10



`MARK=<mark>`

	These parameters make up "fw" filter rules that select traffic for
	each of the classes accoring to firewall "mark". Mark is a decimal
	number packets are tagged with if firewall rules say so. You can
	use multiple `MARK` fields per config.


Note:	Rules for different filter types can be combined. Attention must be
	paid to the priority of filter rules, which can be set below through
	the `PRIO_{RULE,MARK,REALM}` variables.


### Time ranging parameters

`TIME=[<dow><dow>.../]<from>-<till>;<rate>[/<burst>][,<ceil>[/<cburst>]]`
`TIME=60123/18:00-06:00;256Kbit/10Kb,384Kbit`
`TIME=18:00-06:00;256Kbit`

	This parameter allows you to change class bandwidth during the day or
	week. You can use multiple `TIME` rules. If there are several rules with
	overlapping time periods, the last match is taken. The `<rate>`, `<burst>`,
	`<ceil>`, and `<cburst>` fields correspond to parameters `RATE`, `BURST`, 
	`CEIL`, and `CBURST`.

	`<dow>` is single digit from 0 to 6 which represents the day of week as
	returned by `date(1)`. To specify multiple days, just concatenate the
	digits together.



TRIVIAL EXAMPLE
---------------

Consider the following example:
(taken from Linux Advanced Routing & Traffic Control HOWTO)

You have a Linux server with total of 5Mbit available bandwidth. On this
machine, you want to limit webserver traffic to 5Mbit, SMTP traffic to 3Mbit
and everything else (unclassified traffic) to 1Kbit. In case there is unused
bandwidth, you want to share it between SMTP and unclassified traffic.

The "total bandwidth" implies one top-level class with maximum bandwidth
of 5Mbit. Under the top-level class, there are three child classes.

First, the class for webserver traffic is allowed to use 5Mbit of bandwidth.

Second, the class for SMTP traffic is allowed to use 3Mbit of bandwidth and
if there is unused bandwidth left, it can use it but must not exceed 5Mbit
in total.

And finally third, the class for unclassified traffic is allowed to use
1Kbit of bandwidth and borrow unused bandwith, but must not exceed 5Mbit.

If there is demand in all classes, each of them gets share of bandwidth
proportional to its default rate. If there unused is bandwidth left, they
(again) get share proportional to their default rate.

```
Configuration files for this scenario:
---------------------------------------------------------------------------
eth0		eth0-2.root	eth0-2:10.www	eth0-2:20.smtp	eth0-2:30.dfl
----		-----------	-------------	--------------	-------------
DEFAULT=30	RATE=5Mbit	RATE=5Mbit	RATE=3Mbit	RATE=1Kbit
		BURST=15k	BURST=15k	CEIL=5Mbit	CEIL=5Mbit
				LEAF=sfq	BURST=15k	BURST=15k
				RULE=*:80,	LEAF=sfq	LEAF=sfq
						RULE=*:25
---------------------------------------------------------------------------
```

Remember that you can only control traffic going out of your linux machine.
If you have a host connected to network and want to control its traffic on
the gateway in both directions (with respect to the host), you need to setup
traffic control for that host on both (or all) gateway interfaces.

Enjoy.
