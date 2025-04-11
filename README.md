# Hackdonalds
CTF Writeup
Hackdonalds Intigriti CTF Challenge 2025 – A simple writeup 

 

Introduction 

In the 2025 Hackdonalds Intigriti CTF challenge, we encountered a seemingly simple admin login page requiring only a password.  

 

 

 

Our initial approach was to employ tools like cewl to scrape McDonald’s-themed websites for potential “Secret Sauce passwords” and then brute-force the login. However, this initial phase failed to produce any viable credentials.  

 

Reconnaissance and Endpoint Discovery 

Undeterred, we turned to OWASP ZAP to crawl the website and enumerate endpoints. The scan revealed several interesting endpoints in the buildManifest.json file, including: 

/admin, /ice-cream-machines, /ice-cream-details and the .js chunk files that serve those pages.  

  

Notably, an endpoint at /api/parse-xml – more on that later 

   

Despite these discoveries, all of the sensitive endpoints, including /admin and /api/login, were guarded by authentication. Requests to these pages were met with HTTP 307 redirects or 401 Unauthorized responses. 

  

 

Initial Exploitation Attempts 

We experimented with various approaches, such as intercepting and modifying frontend requests using tools like Burp Suite, attempting to trick the backend into treating requests differently. However, these efforts did not yield any success in bypassing the authentication mechanisms. 

  

At this stage, it became clear that a more sophisticated attack vector was required. 

  

 

The Middleware Authorization Bypass (CVE-2025-29927) 

Our breakthrough came when we observed that the version of Next.js in use was vulnerable. Although the CVEs listed for Next.js in ZAP were solely denial-of-service exploits, further research (some google fu) led us to uncover a critical middleware vulnerability, CVE-2025-29927. 

 

Technical Details 

Vulnerability Explanation: 

This vulnerability arises from a flaw in the Next.js middleware responsible for request authentication. When the middleware encountered requests with an unusual header setup, it mistakenly interpreted the request context as an infinite loop of internal subrequests. This misinterpretation led the middleware to bypass the usual authentication checks. 

  

Exploitation: 

By injecting the specially crafted header: 

X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware 

 

 

We confused the middleware into granting access to endpoints that were normally protected. In our tests using Burp Suite, including this header transformed previous 307/401 responses into legitimate 200 OK responses for pages such as /admin and /api/parse-xml. 

  

  

Exploiting the XXE Vulnerability 

With authentication bypassed, our focus shifted to the /api/parse-xml endpoint. This endpoint accepted XML input from the client and leveraged the Node.js library libxmljs2 for XML processing. We see we get a reflected response back from the parser. 

 

 

 

Fortunately for us, libxmljs2 was not configured securely against XXE (XML External Entity) attacks. 

  

How XXE Works 

XML Parsing with External Entities: 

An XML document can include a Document Type Definition (DTD) that defines external entities. If not properly disabled, the XML parser will resolve these entities by retrieving external resources – including arbitrary files on the server. We sent a classic xxe payload for arbitrary file read and boom! 

 

  

Now all that’s left is to find the flag. We used ffuf and a recursive approach to find the flag file after multiple failed attempts. 

 

 

Our Final Payload: 

We crafted an XML payload containing a custom DTD that defined an external entity pointing to a file on the server.  

Sending this payload to the /api/parse-xml endpoint allowed the vulnerable parser to resolve &flag; and return the contents of /app/package.json, which ultimately contained the flag. 

   

Final Exploitation Chain 

Enumeration and Discovery: 

We began with reconnaissance using ZAP, which unearthed hidden endpoints and JavaScript chunks. 

The admin and XML parsing endpoints appeared to be behind authentication. 

Middleware Bypass via CVE-2025-29927 

The exploitation of the Next.js middleware vulnerability by injecting the header 

 XXE Injection 

  

With access to the /api/parse-xml endpoint, we performed an XXE injection using a crafted XML payload. 

  

The payload leveraged the default behavior of libxmljs2 to resolve external entities, allowing arbitrary file disclosure. 

  

After numerous attempts and fuzzing for likely file paths in a Kubernetes environment, the payload reading /app/package.json finally disclosed the flag. 

  

Conclusion 

The Hackdonalds Intigriti CTF challenge demonstrated the critical importance of secure configuration for both middleware components and XML parsers. By exploiting CVE-2025-29927 to bypass authentication and leveraging an XXE vulnerability in a poorly secured XML parser, we achieved arbitrary file disclosure and ultimately retrieved the flag. 

  

Sources and References 

Next.js Middleware Authorization Bypass: Qualys Threat Protection Blog 

Common XXE Vulnerability Techniques and Mitigations: OWASP XXE Prevention Cheat Sheet 

libxmljs2 GitHub Repository: libxmljs2 

  

This multi-step exploitation chain underscores the necessity of defense in depth. Properly configuring XML parsers and robustly validating HTTP headers can help prevent such vulnerabilities from being exploited in production environments. 
