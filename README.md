This code is for a Google+ Sign-In workshop. It takes you through setting up a client side sign in. 

Prerequisites
================================

1. Internet connection
1. Webserver to run the sample from. It wont work from file://
1. Some browser developer tools

On systems with Python installed you can start a server against the local directory with:
```bash
python -m SimpleHTTPServer 8080
```
This will serve the current directory on http://localhost:8080

An alternative if you have no webserver is to paste the contents into a JSBin snippet: http://jsbin.com/

Chrome or Firefox developer tools are convenient, for debugging and seeing the calls back and forth.

Steps
================================

1. Basic Skeleton
-------------------------

For this project we're going to setup a sign-in button, configure it to point to your project, and retrieve the token information from the authentication result. We're then going to use that token to retrieve the user's profile information, and to display a list of their friends. 

To start with, copy step1/index.html to your web root, or paste it into JSBin. This contains a skeleton HTML page, and includes jQuery for user later in the project. Viewing the page should just show "My Google+ Sign-In".

2. Adding The Sign-In Button
-------------------------

The next step is to add the sign-in button to the page, and to create a Javascript function to receive the callback when the sign-in process completes (or fails)!

First we need to add the asynchronous javascript snippet to load the Google+ API from https://developers.google.com/+/web/signin/#step_2_include_the_google_script_on_your_page

```html
<script type="text/javascript">
(function() {
  var po = document.createElement('script');
  po.type = 'text/javascript'; po.async = true;
  po.src = 'https://plus.google.com/js/client:plusone.js';
  var s = document.getElementsByTagName('script')[0];
  s.parentNode.insertBefore(po, s);
})();
</script>
```

Paste this into the code just before the closing body tag. 

Next we need to add in the code for the Sign-In button itself. We need to enter the code below. 

```html
<div id="gPlusButton">
  <button class="g-signin"
      data-requestvisibleactions=""
      data-clientId=""
      data-callback=""
      data-theme="dark"
      data-cookiepolicy="single_host_origin">
  </button>
</div>
```

Add this snippet to your page, just after the &lt;h2&gt;. If you save this, you should now see a Sign-In button!
  
Feel free to try different styles and sizes of the button, see https://developers.google.com/+/web/signin/#sign-in_button_attributes for a whole list.
  
3. Configuring the Sign-In button
-------------------------

At the moment, if you press the button you'll get a pop up with an error of "invalid_request". 

There are a couple of blank parts of that button configuration for you to fill in. 

First is the client ID. To get one of these go to the developer console: https://code.google.com/apis/console/

* Create a new project
* Enabled the Google+ API (switch it to On)
* Go to API Access on the left
* Create an oAuth 2.0 Client ID
* Enter a name, continue, and select web application
* Enter the hostname you're using, with http:// (e.g. jsbin.com or localhost:8080)

This will create a new client ID. Copy the client ID value and paste it into the field in the button: 

```html
data-clientId="XXXXXXXXXX.apps.googleusercontent.com"
```
  
The client ID may also look like

```html
data-clientId="XXXXXXXXXX-XXXXXXXXXXXXXX.apps.googleusercontent.com"
```

If you have more than one.

The second is the data-requestvisibleactions. This will determine what activities your application write. These are a space separated list, like:

```html
"http://schemas.google.com/AddActivity http://schemas.google.com/ReviewActivity"
```

Feel free to choose your own from the list: https://developers.google.com/+/api/moment-types - Add, Buy, CheckIn, Comment, Create, Discover, Listen, Reserve, Review, Want.

Finally, setup your sign in callback. In the script tag near the top of the page with /* Your code goes here! */ create a callback function to receive the result of the sign-in process. It should take a single argument for the authResult. For example: 

```javascript
function onSignInCallback(authResult) {
  console.log("Callback Fired");
}
```
    
You need to reference that in your button, so add your function name in as the callback: 

```html
data-callback="onSignInCallback"
```
    
Now when you reload the page, if you open your developer console you should see the "Callback Fired" option. When you click the button you should be presented with a consent dialogue properly. 

4. Handling the Sign-In callback.
------------------------- 

Next we need to check whether the sign-in actually succeeded. The authResult object has two fields we will need to care about now: access_token and error. If there was an error, then the user has not been signed in. 

If we have an access token, we're good to go - lets hide the sign-in button using jQuery:

```javascript
if (authResult.access_token) {
  $('#gPlusButton').hide();
}
```

We'll probably want to log an error if this happens. Add this code to your function:

```javascript
else if (authResult.error) {
  console.log("Error: " + authResult.error);
  $('#gPlusButton').show();
}
```

If you haven't signed in yet, you'll see that the error you get when not signed in on the initial callback is "immediate_failed". This indicates that seamless immediate sign in failed, so the user will have to approve the application.

Lets finish up by printing out the contents of the access_token. Add the following HTML to your application just below the button:

```html
<div id="authresult"></div>
```
    
In in the access_token part of you onSignInCallback function, populate that with the data from the authResult:

```javascript
$('#authresult').html('Auth Result:</br>')
for (var field in authResult) {
  $('#authresult').append(' ' + field + ': ' + 
                          authResult[field] + '<br />');
}
```
    
Note when you sign in that you consent dialogue includes the app activity types you asked for!

The output from the authResult object includes a number of interesting and useful parameters, such as the ID token object, the scopes you asked for, and the client ID for the application.

5. Retrieving profile and friends
-------------------------

Next, lets retrieve some data with the access token we have. We'll grab the users name and profile picture, and a list of friends they have shared with us. 

First, we'll need to load the gapi client library. This has methods for communicating with the Google REST APIs. We need to tell it to load the Google+ APIs, and give it a callback for what to do once it has:

Under the code to output authResult, still in the access_token block, add the following:

```javascript
gapi.client.load('plus','v1', function(){
  load_profile();
  load_friends();
});
```

As you might guess, next is implementing load_profile and load_friends! Add some HTML blocks to contain the information above the authresult div in the body of the page. 

```html
<div id="profile"></div>
<div id="people"></div>
```

First, lets retrieve the users profile information. Under the onSignInCallback function, still inside the script tag, add a function that calls the gapi.client.plus.people.get method. All the Google+ API functions are under the 'plus' space, and are named resource.method - e.g. people resource, get method. We'll pass in the special string 'me', which represents the current user. 

```javascript
function load_profile() {
  var request = gapi.client.plus.people.get( {'userId' : 'me'} );
  request.execute( function(profile) {
    $('#profile').empty();
    $('#profile').append(
        $('<p><img src=\"' + profile.image.url + '\"></p>'))
    $('#profile').append(
        $('<p>Hello ' + profile.displayName + '!<br />Tagline: ' +
        profile.tagline + '<br />About: ' + profile.aboutMe + '</p>'));
  });
}
```

We should see the user's profile picture and some details from their profile (if they're public). We can also load the users friends, but making a similar call to plus.people.list. This time we need the user ID again, but also a "collection". That defines which list of people we want. For Google+ Sign-In that is always "visible" - the people shared with the application.

```javascript
function load_friends() {
  var request = gapi.client.plus.people.list({
    'userId': 'me',
    'collection': 'visible'
  });
  request.execute( function(people) {
    $('#people').empty();
    for (var personIndex in people.items) {
      person = people.items[personIndex];
      $('#people').append('<img src="' + person.image.url + '">');
    }
  });
}
```
    
6. That's it!
-------------------------

You now have signed in, and retrieved the users details. There are more Quickstarts which show how to do this in a number of languages up on the Google+ Developers page. There's also a whole lot of documentation, and examples of how to implement each of the other features. 

https://developers.google.com/+/