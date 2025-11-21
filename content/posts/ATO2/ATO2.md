---
title: "Fully account take over; they printed the password"
date: 2025-11-21T18:04:12+03:00
draft: false
---
## Intro
**Vulnerability Type:** Information Disclosure & Broken Authentication Logic

I was just browsing :), I came across an login page in a website is often I visit. they updated the pages in their websites, so I tested their new updates. the problem was in the **"Forget password"**. While analyzing the authentication workflow in JavaScript and the API responses I identified a two-stage vulnerability chain.

When you press **"Forget password"** it only ask you about the username, if you enter it, it will send a temporary password to the phone number and the email account that linked to it.

## First Vulnerable Code Snippet

So I start with the source code in the page. trying to understand how it work

```javascript
// Vulnerable Code Snippet
$.ajax({
    type: "POST",
    url: '/SEND/Code',
    data: { email: email }, // here we put the username to send to the server
    success: function (data) {
        const myHcode = document.getElementById("Hcode");
        if (data && data.success) {
            // CRITICAL FLAW: The server sends the 'scode' (OTP) back to the client!
            if (myHcode) myHcode.value = data.scode;
            alert('Password sent to your email.');
        }
        // ...
    }
});
```
it send to the server my username checking if it is in the database, then it send back to my browser **"Hcode"**  which is the OTP!

the behavior is:
- Sends `email=<UserName>` (they use “email” as a param name but it’s really the username).
- On success:
    - Writes **`data.scode` into a hidden field `Hcode`**.
    - Shows “Password sent to your email.”
    - Redirects to `/Login/Logout`.

the response will be like this:
I intercepted the request to the /SEND/Code endpoint using Burp Suite.
- **Request:** `POST /SEND/Code` with `email=TARGET_ID`
- **Response:** The server responded with a JSON object containing the valid OTP.

```json
{
  "success": true,
  "scode": "9482"  // <--- The Secret OTP Leaked Here
}
```

we did not finish here. when we reach this stage it actuality sending password to the username that we put. we need to get this password from the victim "Normally we can't see he's phone messages or access he's email to see the password". but we find the OTP for now.
## Second Vulnerable Code Snippet

**this JS snippet code is the big problem, it can show up the password for you if you give him two things: the username + OTP code that is leaked in the previous snippet code**

```JS
function checkCode() {
    const myHcode = document.getElementById("Hcode");
    const enteredCode = $('#authCode').val();
    const emailInput = document.getElementById('forgotEmail');
    const email = emailInput?.value || '';
    const concode = myHcode ? myHcode.value : '';

    $.ajax({
        type: "POST",
        url: '/SEND/ConfirmationCode',
        data: { enteredCode: enteredCode, confirmationCode: concode, email: email },
        success: function (data) {
            if (data && data.success) {
                alert('Your password reset. Login again with new password: ' + data.newpass); // this is main thing that we need
                window.location.href = '/Login/Logout';
            } else {
                alert('Code is not valid. Please try again.');
            }
        },
        error: function () {
            alert('An error occurred. Please try again.');
        }
    });
}
```

The first lines is the variables, saving the OTP in `myHcode` then it save it to `concode`. `enteredCode` will be OTP also. all of these the server and the client will do it us without interacting
We need to catch this line:
`alert('Your password reset. Login again with new password: ' + data.newpass);`

**How can we print it?**

Let's use this endpoint that is in JS code above: 
```js
        type: "POST",
        url: '/SEND/ConfirmationCode'
```

to reach this line: `alert('Your password reset. Login again with new password: ' + data.newpass);`

you have to create a new request in Burp suite with that endpoint and putting the parameters that is 
`enteredCode=1111&confirmationCode=1111&email=UserName`
1. The Server receives this request.
2. The Server validates the enteredCode matches the confirmationCode (or the session).
3. The Server generates the JSON response containing newpass.
## Combine the two vulnerable code (testing)

Now let's say we have `n4if` user we went to "Forget password" and type `n4if` then we click "send password" it will generate a new temporary password and send it to the phone and email

![](/posts/ATO2/1.png)

OTP that the server exposed is: `3074`
**it should not show this to any requested user**

Taking this OTP and use it to next endpoint that is in the second JS code,
We created this request based in JS code.

![](/posts/ATO2/2.png)

password is:
```json
{"success":true,"newpass":"Reset@8705Wt"}
```

## Conclusion & POC Summary

By analyzing the JavaScript, we identified that the application trusts the client to handle the secrets.

1. **Information Leak:** We captured the OTP in the response of SendCode.
2. **Broken Logic:** We utilized the CheckConfirmationCode endpoint to feed that stolen OTP back to the server.
3. **Critical Failure:** The server returned the credentials in clear text, allowing for immediate unauthorized access.
**Recommendation:**  
The logic must be moved entirely to the server side. The client should never handle scode (the correct code) or newpass (the resulting password).




