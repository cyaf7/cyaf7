# View Disk & User Info

Who is logged in:

{% hint style="info" %}
who   # Shows all logged-in users\
last   # Shows login history\
w      # Shows who is logged in and what they are doing\
finger     # More info about users
{% endhint %}

{% columns %}
{% column %}
**command**

sudo dmidecode

sudo fdisk -l

filename /etc/sudoers
{% endcolumn %}

{% column %}
**purpose**

Shows hardware/system info

Shows disk size and partitions

Shows file type
{% endcolumn %}
{% endcolumns %}
