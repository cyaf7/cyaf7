# System Monitoring

To monitor disk space, memory, network, and system behavior — **important for detecting performance or security issues**.

{% columns %}
{% column width="25%" %}
command

top

df -h

dmesg

iostat

ip route

ss | more

free

cat /proc/meminfo
{% endcolumn %}

{% column %}
what it does

Live view of system resources

Show disk space usage (in human format)

Kernel and boot messages

CPU and disk I/O statistics

Shows routing and network

Inspect open sockets

Physical and virtual memory usage

Detailed memory info | |
{% endcolumn %}
{% endcolumns %}
