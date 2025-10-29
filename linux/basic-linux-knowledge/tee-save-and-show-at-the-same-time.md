# tee — Save and Show at the Same Time

Use when you want to **see the output on screen** and **save it to a file** at the same time.

{% hint style="info" %}
ls -l | **tee** list.txt
{% endhint %}

You see the list, and it's also saved in list.txt.

{% hint style="info" %}
ls -l | tee -a log.txt
{% endhint %}

Same, but appends to log.txt instead of overwriting it.

