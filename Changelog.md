VERSION HISTORY
===============

0.8.6
-----

  * Sławomir Paszkiewic
    - move -maxdepth before non-option arguments
      to stop "find" from emitting warnings
  * Oleksandr Darchuk
    - fix parsing of PRIO_{*} keywords in class file
  * Vasili Shtyk
    - fix to avoid exiting with exit code 1 on "start"
  * ankry
    - add (empty) status and force-reload commands
      for LSB compliance

0.8.5
-----

  * Nathan Shafer <nicodemus at users.sourceforge.net>
    - allow symlinks to class files
  * Seth J. Blank <antifreeze at users.sourceforge.net>
    - replace hardcoded ip/tc location with variables
  * Mark Davis <mark.davis at gmx.de>
    - allow setting of PRIO_{MARK,RULE,REALM} in class file

0.8.4
-----

  * Lubomir Bulej <pallas at kadan.cz>
    - fixed small bug in RULE parser to correctly parse
      rules with identical source and destination fields
    - removed the experimental INJECT keyword
    - ignore *~ backup files when looking for classes
  * Mike Boyer <boyer at administrative.com>
    - fix to allow arguments to be passed to "restart" command
  * <face at pos.sk>
    - fix to preserve class priority after timecheck

0.8.3
-----

  * Lubomir Bulej <pallas at kadan.cz>
    - use LC_COLLATE="C" when sorting class files
  * Paulo Sedrez
    - fix time2abs to allow hours with leading zero in TIME rules

0.8.2
-----

  * Lubomir Bulej <pallas at kadan.cz>
    - thanks to Hasso Tepper for reporting the following problems
    - allow dots in interface names for use with VLAN interfaces
    - fixed a thinko resulting from "cosmetic overdosage" :)

0.8.1
-----

  * Lubomir Bulej <pallas at kadan.cz>
    - added function alternatives for sed/find with less features. To
      enable them, you need to set HTB_BASIC to nonempty string.
    - added posibility to refer to RATE/CEIL of parent class when
      setting RATE/CEIL for child class. Look for "prate" or "pceil"
      in the documentation.
    - fixed broken "timecheck" invocation

0.8.0
-----

  * Lubomir Bulej <pallas at kadan.cz>
    - simplified and converted CBQ.init 0.7 into HTB.init
    - changed configuration file naming conventions
    - lots of HTB specific changes
