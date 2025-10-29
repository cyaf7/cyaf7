# Signals & Killing Processes

Signals are short messages sent to processes by the OS to **interrupt**, **kill**, or **control** them.

examples:

{% hint style="info" %}
kill -SIGINT PIDnumberoftask    # interrupt process\
kill -SIGKILL PIDnumberoftask    # force kill
{% endhint %}

Common signals:

{% columns %}
{% column %}
**signal**

SIGINT

SIGTERM

SIGKILL

SIGSTOP

SIGCONT

SIGSEGV
{% endcolumn %}

{% column %}
**meaning**

Interrupt (Ctrl+C)

Graceful termination

Forcefully kill (can’t be ignored)

Stop a process

Resume a stopped process

Segmentation fault
{% endcolumn %}
{% endcolumns %}
