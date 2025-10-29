# cut — Cut Specific Fields (Columns)

Splits lines using a **separator** and shows specific columns.

it can extract parts of a structured file (like CSVs or /etc/passwd) and it's faster and simpler than awk for simple tasks.

-f1 = show first field

-d ':' = use colon as separator

Shows all usernames in the system:

{% hint style="info" %}
cut -d ':' -f1 /etc/passwd
{% endhint %}

Shows the second column of a CSV:

{% hint style="info" %}
cut -d ',' -f2 employees.csv
{% endhint %}
