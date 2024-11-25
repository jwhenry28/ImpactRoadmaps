If we cannot hit any interesting internal services, we can try to enumerate interesting data from the HTTP requests issued by the vulnerable server.

This Impact Technique has one step:
1. Issue a request to Burp Collaborator and take note of any useful insights

# 1. Issue a request to Burp Collaborator and take note of any useful insights
In Burp Suite, click the "Collaborator" tab and copy a new domain name. Send this domain to the vulnerable server in your SSRF payload. You should get a ping back in the Collaborator window:

Review the Collaborator logs and the data from the inbound HTTP request. Think about what you can infer from it. 

Sometimes, there will be obvious conclusions:
![[Pasted image 20240208104805.png]]
- In this example, there is all kinds of interesting data in the request's HTTP headers. You know the version of Python, the HTTP client used, and loads of Kubernetes data.

Other times, your request might be a little more bare-bones:
![[Pasted image 20240208104915.png]]
- In this example, we can still use the IP address of the inbound request to determine the cloud provider of the application and limited information about the tech stack (i.e., the application runs from an EC2 instance.)
- If this EC2 instance sat behind a load balancer and was not directly accessible to the internet, learning the IP address could be useful information.