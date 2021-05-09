---
layout: single
title: About
permalink: /about/
author_profile: true
---

I'm mostly interested in web application security, but dabble in a bit of everything.
## Background
Linux systems administration coupled with some Django/python development work. Did everything from physical server racking/installs to hosting public websites, and all the load balancing/DNS/containerization/dev work in between. I did that for about a year and a half, got OSCP, and a few months later I moved into a penetration testing role.
## Certifications
[Offensive Security Certified Professional (OSCP)](https://www.offensive-security.com/pwk-oscp/): [Verify](https://www.youracclaim.com/badges/2352083f-2957-4c8d-88db-5df6279b61da)  
[Offensive Security Web Expert (OSWE)](https://www.offensive-security.com/awae-oswe/): [Verify](https://www.youracclaim.com/badges/ab2933bd-5373-443f-b6fe-6a0608ecbe6b)
## Talks/Podcats/etc

|  Talk  |  Org  | Host |
| :---: | :---------------: | :---: |
[From Veteran to Penetration Tester](https://www.youtube.com/watch?v=DUdjVuHE8tk) | [Offensive Security Interviews](https://www.offsecinterviews.com/) | [Jon Helmus](https://twitter.com/Moos1e_Moose)
[Hardware Hacking: The Easy Way In...](https://www.youtube.com/watch?v=MnyRjA-9WQM) | [The Pwn School Project](https://pwnschool.com/) | [Phillip Wylie](https://twitter.com/PhillipWylie)
[Interview With A Red Teamer - Cory Billington](https://www.youtube.com/watch?v=L1v4CBr_IOw) | [cwinfosec](https://www.youtube.com/channel/UCFSN5ly66Lw_H_R4uX2cF0A) | [cwinfosec](https://twitter.com/cwinfosec)

## Tools
### SharpFind
[SharpFind](https://github.com/mcorybillington/SharpFind) is a tool written to provide some of the useful features of the Unix tool `find`, such as writable files, recently modified files, wildcard searching, and it can identify .NET assemblies. Since it is written in .NET, you can use it over C2 in Cobalt Strike with `execute-assembly` or Covenant using `Assembly`.
### sshspray
[sshspray](https://github.com/mcorybillington/sshspray) is a multi-threaded python tool that can be used to spray ssh keys or passwords across a large number of hosts. I wrote this application as [hydra](https://en.wikipedia.org/wiki/Hydra_(software)) did not have the capability to provide an SSH key.


