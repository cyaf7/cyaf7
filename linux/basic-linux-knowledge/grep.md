# grep

Reads a file **line by line** and shows only the lines that **match a specific word or pattern**.

{% hint style="info" %}
grep word filename
{% endhint %}

Finds a user in the system’s password file (used to check if a username exists, and see similar ones):

{% hint style="info" %}
grep username /etc/passwd
{% endhint %}

Counts how many times the word appears in the file:

{% hint style="info" %}
grep -c word filename
{% endhint %}

Shows all lines that **don’t contain** the word:

{% hint style="info" %}
grep -v word filename
{% endhint %}

