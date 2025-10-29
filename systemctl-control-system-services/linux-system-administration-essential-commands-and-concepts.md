# Linux System Administration: Essential Commands & Concepts

### **Systemd and Service Management**

systemctl -> Controls system services

{% hint style="info" %}
systemctl start|stop|status servicename.service
{% endhint %}

{% hint style="info" %}
systemctl enable|restart|reload servicename.service
{% endhint %}

Example:&#x20;

{% hint style="info" %}
systemctl stop firewalld.service
{% endhint %}

**List all units:**&#x20;

{% hint style="info" %}
systemctl list-units --all
{% endhint %}

**To add a service:**

Create a unit file in&#x20;

{% hint style="info" %}
/etc/systemd/system/servicename.service
{% endhint %}

**System shutdown/reboot:**

systemctl reboot

systemctl poweroff

### **System Monitoring**

**Processes and Usage**

top – live view of CPU/memory/processes

free – shows RAM (physical + virtual)

df -h – disk usage in human-readable form

**Hardware and System Logs**

dmesg – kernel messages

iostat – CPU/disk I/O statistics

**Network**

ip route | column -t – formatted routing table

ss | more – investigate sockets

cat /proc/meminfo – detailed memory info

### **Process Management**

Ctrl + Z – send process to background

jobs – view background processes

fg – bring background job to foreground

nohup \<command> – run process immune to hangup

pkill \<process> – kill by name

kill -SIGINT \<PID> → interrupt (doesn’t kill vi)

kill -SIGKILL \<PID> → force kill

### **Log Monitoring**

Log directory: /var/log/

Key logs:

boot.log – system boot events

secure – authentication & security (tail -f /var/log/secure)

message – hardware, app, and general logs

cron, maillog, httpd, etc.

Use **grep -i "error"** for filtering logs.

### **System Maintenance**

shutdown, reboot, halt

init 0 – shutdown, init 6 – reboot

init 3 – multi-user mode

