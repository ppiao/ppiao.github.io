---
layout: post
title: strace GSoC 2017 netlink socket parsers
---

# Describe my work briefly

* Merge the code by GSoC 2016.
* Extend to support more netlink family protocols.

# What is done

* Fixed and merged the code of general netlink attribute decoder by GSoC 2016.
* Fixed and merged the code of decoding NETLINK_ROUTE messages by GSoC 2016.
* Fixed and merged the code of decoding NETLINK_SOCK_DIAG messages by GSoC 2016.
* Implemented decoding of NETLINK_SOCK_DIAG netlink attributes.
* Implemented decoding of netlink message ack flags.
* Implemented decoding of nlmsgerr netlink attributes.
* Implemented protocol specific decoding of NETLINK_SELINUX.
* Implemented protocol specific decoding of NETLINK_CRYPTO.
* Implemented basic decoding of NETLINK_KOBJECT_UEVENT.

# TODO

* Read the code, fix the bug. "Given enough eyeballs, all bugs are shallow".
* Extend to support more netlink family protocols.
* Further decode some fields, such as cru_type, cru_mask and cru_flags.
* <span style="text-decoration: line-through">Further decode NETLINK_KOBJECT_UEVENT. (libudev and kernel messages.)</span>
* Extend or redesign general netlink attribute decoder for decode attribute that
  depend on other attribute.

# Others

* [My strace GSoC 2017 Journey](/gsoc-2017)
* [Contribution within the GSoC 2017][GSoC-commit]
* [JingPiao Chen's strace GSoC 2017 status report][status-report]

**Useful link:**

* [strace GSoC 2016: Netlink socket parsers][GSoC-2016]
* [Netlink overview and its strace parsers][strace-netlink]

[GSoC-commit]: https://github.com/strace/strace/commits?author=ppiao&since=2017-05-01&until=2017-08-31
[status-report]: https://sourceforge.net/p/strace/mailman/search/?q=+JingPiao+Chen%27s+GSoC+status+report
[GSoC-2016]: https://summerofcode.withgoogle.com/archive/2016/projects/5126179027681280
[strace-netlink]: http://blog.saruta.eu/netlink_strace.html
