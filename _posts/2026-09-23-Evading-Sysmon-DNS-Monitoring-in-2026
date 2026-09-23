**Revisiting a 2019 Technique Against Modern Sysmon**

Back in 2019, XPN published *[Evading Sysmon DNS Monitoring](https://blog.xpnsec.com/evading-sysmon-dns-monitoring/)*, exploring how Sysmon 10.1 collected DNS telemetry through ETW and how that telemetry could be affected from user mode.

More than seven years later, I wanted to revisit the same research on a modern Windows environment with the latest version of Sysmon.

This post is not a reproduction of the original research. Instead, it looks at how the underlying behavior has changed, what still works, what no longer does, and what can be learned by tracing the modern Windows DNS stack and Sysmon's telemetry path.

> **Modern research:** Windows 11 + Sysmon 15.22 (2026)
