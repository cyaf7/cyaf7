# awk — Smart Field Processing

Processes structured text line by line, splitting into **fields** automatically.

Shows the first word from every line:

{% hint style="info" %}
awk '{print $1}' names.txt
{% endhint %}

Shows usernames and their shell:

{% hint style="info" %}
awk -F: '{print $1, $7}' /etc/passwd
{% endhint %}

Shows device name and disk usage %:

{% hint style="info" %}
df -h | awk '{print $1, $5}'
{% endhint %}

{% columns %}
{% column %}
command

$0

$1,$2,$3
{% endcolumn %}

{% column %}
result

whole line

1st, 2nd, 3rd column
{% endcolumn %}
{% endcolumns %}
