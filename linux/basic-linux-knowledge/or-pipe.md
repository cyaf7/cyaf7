# | — Pipe

Use to send the output of one command into another.

{% hint style="info" %}
cat names.txt **|** sort | uniq
{% endhint %}

Take the names, sort them, and remove duplicates.

{% hint style="info" %}
ls -l **|** grep ".sh"
{% endhint %}

Only show .sh scripts in a detailed list.
