---
layout: page
title: FreeIPA Notes
permalink: /notes/freeipa-troubleshooting-notes
---

* What does `/var/log/secure` say about our user on the target machien?


* Can we get the users information?

```
getent passwd user@domainfqdn
```

* Is the FreeIPA server Up and running as we'd expect?

```
ipactl status
```

* Can FreeIPA find any users?

```
ipa user-find --all
```
