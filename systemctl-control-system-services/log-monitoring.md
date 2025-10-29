---
description: command *last*
---

# Log Monitoring

Logs are **essential in cybersecurity** for auditing and intrusion detection.

{% columns %}
{% column width="41.66666666666667%" %}
**file/command**

/var/log/

/var/log/boot.log

/var/log/secure

tail -f /var/log/secure

/var/log/messages

grep -i error /var/log/...

httpd
{% endcolumn %}

{% column %}
**meaning**

Main log directory

Boot messages

Auth & security logs

Real-time monitoring of auth logs

Kernel, hardware, app messages

Find errors in logs

Apache web server logs
{% endcolumn %}
{% endcolumns %}
