If you can issue requests to internal IPs/ports, you should try to scan for internal services. 

This Impact Technique has two steps:
1. Launch Burp Intruder against a wordlist of internal IP addresses and observe the responses.
2. Attempt to interact with any identified services

# 1. Launch Burp Intruder against a wordlist of internal IP addresses
You can use the following Python script to generate a list of all internal IP addresses:
```python
import ipaddress

local = "127.0.0.1"
classAs = "10.0.0.0/8"
classBs = "172.16.0.0/12"
classCs = "192.168.0.0/16"

ports = []#["80", "443"] # <-- ADJUST AS NEEDED

classes = [classAs, classBs, classCs]

print(local)
for c in classes:
    for ip in ipaddress.IPv4Network(c):
        if ports:
            for p in ports:
                print(str(ip) + ":" + str(p))
        else:
            print(ip)

```
- Update `ports` as needed if you wish to scan particular ports
	- To generate all TCP ports: `ports = [x for x in range(0, 65536)]`
- If you have source code, try grepping in the code base for `https?://.*` to identify hard-coded service hostnames. Add these to the top of your list. 

Run your scan. Depending on how many IP/port combinations you have, it may take a very long time. 
If you are having trouble with the application blocking your requests to hit an internal IP address, see [[Exfiltrate Cloud Metadata#Bypassing SSRF Blacklists]].

Once the scan is finished, observe the responses. If your SSRF is not blind, you will hopefully have a few valid HTTP responses. Observe what they contain, and see if you can identify the services by Googling any interesting keywords. If you can identify specific services, move to step 2 below.

If your SSRF is blind, try to infer which IP/port combinations corresponded to active services based on side channels like the status code, error message, or request duration. For example:
![[Pasted image 20240208103923.png]]

In this case, the engineer could not read the responses received by their HTTP request. However, based on the status code, they could infer a service ran on port 80. And based on the duration of the request ("Response received"), they could also infer services ran on ports 3389 and 135. By assuming default ports, we could also infer this machine ran an HTTP server, RPC, and RDP. Further, because these services are typically associated with Windows, we could also assume this host is a Windows machine. Any conclusions you can reasonably assume are worth including in your writeup. 

# 2. Attempt to interact with any identified services
Once you've identified internal IP addresses that run active services, you can try to issue legitimate requests to those services. You will likely need to identify what the service is. This will be very hard to do with a blind SSRF unless you find a hard-coded service URL elsewhere in the engagement and can reasonably assume what service is running on that host based on the hostname alone. 

If you can identify the service (e.g., Jenkins, JFrog, etc.), the best thing to do next is consult the service's documentation. You want to find supported endpoints that you can access with your SSRF payload. Typically, this will mean unauthenticated GET requests, unless you have additional control over the HTTP request. 
- **NOTE**: "heartbeat" or "status" endpoints are often great targets for this step, as they are frequently unauthenticated GET requests and will not accidentally crash the service or leak sensitive data.

This step is intentionally open-ended. You will need to think on your own about how to interact with the service. 