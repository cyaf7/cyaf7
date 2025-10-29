# Wildcards in Linux

* [ ] **"\*" -**> Matches **zero or more characters**.
* [ ] "?" -> Matches **exactly one character**.
* [ ] \[] -> Matches **any one character inside the brackets** (range or list).

{% hint style="info" %}
touch abc{1..9}-XYZ
{% endhint %}

creates all these 9 files:&#x20;

abc1-XYZ abc2-XYZ ... abc9-XYZ

And:

{% hint style="info" %}
rm _YX_
{% endhint %}

Removes all files that have “YX” anywhere in the name.
