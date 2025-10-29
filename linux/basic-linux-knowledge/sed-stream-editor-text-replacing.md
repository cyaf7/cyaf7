# sed - Stream editor (text replacing)

Edits text inside files **automatically**. It can replace words, delete lines, or remove empty spaces.

Example to replace a word (not saved permanently)::

{% hint style="info" %}
sed 's/name/othername/g' filename
{% endhint %}

Replace and save permanently::

{% hint style="info" %}
sed -i 's/name/othername/g' filename
{% endhint %}

Remove a word (but leaves empty space):

{% hint style="info" %}
sed -i 's/word\_to\_remove//g' filename
{% endhint %}

