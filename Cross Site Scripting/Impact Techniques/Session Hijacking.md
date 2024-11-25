If the target user(s) have the privileges to perform other interesting actions in the application, you can hijack their session to perform those actions. This is very similar to cross-site request forgery. The user will trigger your payload, which will issue additional requests to the backend on the user's behalf. 

This Impact Technique has two steps:
1. Identify a sensitive endpoint that the user can access.
2. Issue a request to the sensitive endpoint with your XSS payload.

# 1. Identify a sensitive endpoint
By now, you've probably performed a walkthrough of the application and should have a good idea of all the features available through the application. Your XSS payload will land on a page that is accessible to certain users. Think about what permissions these users are likely to have and what endpoint their permissions will allow them to access. Pick one or more endpoints that perform some interesting functionality, such as:
- Changing the user's password
- Changing the user's 2FA method
- Adding a new user to the application/tenant
- Removing a user from the application/tenant
- Uploading sensitive data
- Downloading sensitive data
- Etc.

If you cannot locate anything that the target user has sufficient privileges for, consider the following:
1. If you have source code, determine how the application routes URLs. It may be helpful to enumerate all of the URL handlers on the backend to see if you missed anything earlier.
2. In Burp, look through "Target" > "Site map" > (click on your application's domain) to see all of the endpoints Burp has observed. You may have missed something that Burp recorded.

After you've identified a sensitive endpoint, proceed.

# 2. Issue a request
If you've already captured the request in Burp, you can easily generate a payload like so:
![[Pasted image 20240206133825.png|750]]

If you have not captured the request in Burp and cannot figure out where the request is triggered in the frontend (likely because you identified the endpoint by looking at the source code), you can attempt to recreate the request with JavaScript's [fetch API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API).