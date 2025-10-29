# Combining and Splitting Files

Combine multiple files into one, or split large files into smaller chunks.

Combine files:

{% hint style="info" %}
cat file1 file2 file3 > combined.txt
{% endhint %}

Split a file:

{% hint style="info" %}
split file.txt
{% endhint %}

Split by number of lines will split file.txt into pieces with 300 lines each. The output files start with fileoutput\_.

{% hint style="info" %}
split -l 300 file.txt fileoutput\_
{% endhint %}

