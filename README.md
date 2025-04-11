# Hackdonalds

Intigriti CTF Challenge 2025 CTF Writeup

At the start of the challenge, we are presented with a simple admin login page requiring only a password.  

 ![image](https://github.com/user-attachments/assets/f9f29de0-22b8-46e8-bcab-7c8dcd8d076c)


Our initial approach was to employ cewl to scrape hackdonalds and other McDonald’s-themed websites for potential “Secret Sauce passwords”. We used hashcat to generate mutations, and then attempted to brute-force the login. However, this initial phase failed to produce any viable credentials.  


Reconnaissance and Endpoint Discovery 

Undeterred, we turned to OWASP ZAP to crawl the website and enumerate endpoints. The scan revealed several interesting endpoints in the buildManifest.json file, including: 

/admin, /ice-cream-machines, /ice-cream-details and the .js chunk files that serve those pages.  

Notably, an endpoint at /api/parse-xml – more on that later 

![image](https://github.com/user-attachments/assets/f6e4db69-9165-4288-b68e-3d24af914b26)
![image](https://github.com/user-attachments/assets/5315b9e2-289a-4e8d-b995-9f0f187522d5)


Despite these discoveries, all of the sensitive endpoints, including /admin and /api/login, were guarded by authentication. Requests to these pages were met with HTTP 307 redirects or 401 Unauthorized responses. 

  
![image](https://github.com/user-attachments/assets/b30cae0e-d8df-4e19-a972-c47a1959b440)

 

Initial Exploitation Attempts 

We experimented with various approaches, such as intercepting and modifying frontend requests using tools like Burp Suite, attempting to trick the backend into treating requests differently. However, these efforts did not yield any success in bypassing the authentication mechanisms. 

At this stage, it became clear that a more sophisticated attack vector was required. 

![image](https://github.com/user-attachments/assets/578057f9-353c-41ba-a3e4-00b6fe3cfd76)
  

The Middleware Authorization Bypass (CVE-2025-29927) 

Our breakthrough came when we observed that the version of Next.js in use was vulnerable. Although the CVEs listed for Next.js in ZAP were solely denial-of-service exploits, further research (some google fu) led us to uncover a critical middleware vulnerability, CVE-2025-29927. 

Vulnerability Explanation: 

This vulnerability arises from a flaw in the Next.js middleware responsible for request authentication. When the middleware encountered requests with an unusual header setup, it mistakenly interpreted the request context as an infinite loop of internal subrequests. This misinterpretation led the middleware to bypass the usual authentication checks. 



![Screenshot 2025-04-11 161532](https://github.com/user-attachments/assets/13f746e8-e0f2-4786-8e23-2e1e856dd55c)


Exploitation: 

By injecting the specially crafted header: 
X-Middleware-Subrequest: middleware:middleware:middleware:middleware:middleware 

We confused the middleware into granting access to endpoints that were normally protected. In our tests using Burp Suite, including this header transformed previous 307/401 responses into legitimate 200 OK responses for pages such as /admin and /api/parse-xml. 

  
![Screenshot 2025-04-11 161646](https://github.com/user-attachments/assets/db4fa2aa-0d63-4dd1-a184-f8b70b63243b)

  

Exploiting the XXE Vulnerability 

With authentication bypassed, our focus shifted to the /api/parse-xml endpoint. This endpoint accepted XML input from the client and leveraged the Node.js library libxmljs2 for XML processing. We see we get a reflected response back from the parser. 


 ![Screenshot 2025-04-11 161719](https://github.com/user-attachments/assets/efc84bd7-bd9e-4f57-8f62-5a51f84d19f8)
 

Fortunately for us, libxmljs2 was not configured securely against XXE (XML External Entity) attacks.For the unintiated - this is how XXE Works-
An XML document can include a Document Type Definition (DTD) that defines external entities. If not properly disabled, the XML parser will resolve these entities by retrieving external resources – including arbitrary files on the server. We sent a classic xxe payload for arbitrary file read and boom! 

 
![Screenshot 2025-04-11 161804](https://github.com/user-attachments/assets/816d32a5-e214-4fd4-aa2d-facb9326e80a)

  

Now all that’s left is to find the flag. We used ffuf and a recursive approach to find the flag file after multiple failed attempts. 

 
![Screenshot 2025-04-11 161851](https://github.com/user-attachments/assets/ff22a964-dda3-4035-ab31-82c6ccb764b7)

 

Our Final Payload: 

We crafted an XML payload containing a custom DTD that defined an external entity pointing to a file on the server.  
Sending this payload to the /api/parse-xml endpoint allowed the vulnerable parser to resolve &flag; and return the contents of /app/package.json, which ultimately contained the flag. 

![Screenshot 2025-04-11 162040](https://github.com/user-attachments/assets/f086ffd9-dc77-49d4-9322-c4a0b393ff63)

Conclusion:
The Hackdonalds Intigriti CTF challenge demonstrated the critical importance of secure configuration for both middleware components and XML parsers. By exploiting CVE-2025-29927 to bypass authentication and leveraging an XXE vulnerability in a poorly secured XML parser, we achieved arbitrary file disclosure and ultimately retrieved the flag. This multi-step exploitation chain underscores the necessity of defense in depth. Properly configuring XML parsers and robustly validating HTTP headers can help prevent such vulnerabilities from being exploited in production environments. 
