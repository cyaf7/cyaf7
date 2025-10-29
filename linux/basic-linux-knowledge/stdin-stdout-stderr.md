# stdin, stdout, stderr

In Linux, every command can:

* **take input** (stdin)
* **show output** (stdout)
* **show error messages** (stderr)

You can **redirect** these to or from files or commands.

### **stdin (Standard Input):**

Use when you want a command to read input from a **file** instead of the keyboard.

{% hint style="info" %}
cat < hello.txt&#x20;
{% endhint %}

Instead of typing text into cat, it reads it from hello.txt.

### **stdout (Standard Output)**

Use when you want to **save the result** of a command to a file instead of seeing it on the screen.

{% hint style="info" %}
ls > files.txt
{% endhint %}

Saves the list of files into files.txt. Nothing appears on screen.

### **stderr (Standard Error)**

Use when you want to capture only the error messages of a command.

{% hint style="info" %}
ls no\_such\_file 2> errors.txt
{% endhint %}

