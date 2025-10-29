# /etc/login.defs + chage — Managing password and account expiration

Security teams use this to **enforce password expiration**, lock inactive users, and define **account policies**.

**File: /etc/login.defs**

This file contains default password settings like:

{% hint style="info" %}
PASS\_MAX\_DAYS 9999      # Max days before password expires\
PASS\_MIN\_DAYS 0              # Min days before changing password again
{% endhint %}

Command: chage — Change user aging information

{% hint style="info" %}
chage -m 5 -M 90 -W 10 -I 3 username
{% endhint %}

{% columns %}
{% column %}
option

-m

-M

-w

-i
{% endcolumn %}

{% column %}
meaning

Minimum days before changing

Maximum days a password is valid

Warning days before expiration

Inactive days before locking
{% endcolumn %}
{% endcolumns %}
