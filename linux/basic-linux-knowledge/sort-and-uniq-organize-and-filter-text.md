# sort and uniq — Organize and filter text

sort arranges lines alphabetically or numerically:

{% hint style="info" %}
sort filename&#x20;
{% endhint %}

and sort in reverse as well:

{% hint style="info" %}
sort -r filename
{% endhint %}

uniq removes duplicates (often used together!).

Always use sort before uniq because uniq only removes consecutive duplicates.

{% hint style="info" %}
sort file | uniq -c
{% endhint %}
