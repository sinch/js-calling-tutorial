# Using Sinch JS SDK to call a phone number
In this tutorial you will build an app to call a regular phone number from your browser. This is a pretty sweet feature that can be used, for example, to enable your users to call hotels in a different country cheaply and conveniently.
 
Application flow:
Flow A: Register user -> place a call **or** 
Flow B: Login -> place a call

## Setup
1. If you don’t have a Sinch developer account, please sign up and register a new app at 
[www.sinch.com/signup](http://www.sinch.com/signup)
2. Download the SDK from [www.sinch.com/js-sdk](www.sinch.com/js-sdk "Download JS-SDK")

Create an index.html file with references to jQuery and the Sinch JavaScript SDK. Also, to get some basic css, you will use bootstrap.

```html
<!DOCTYPE html>
<html>
<head>
    <title>Call sinch</title>
    <script src="https://ajax.googleapis.com/ajax/libs/jquery/1.11.1/jquery.min.js">
    </script>
    <script src="sinch.min.js"></script>
    <link rel="stylesheet" 
    href="https://maxcdn.bootstrapcdn.com/bootstrap/3.2.0/css/bootstrap.min.css">
</head>
<body>
```


## Adding UI for Login and User Registration
You don't want anonymous users to be able to just go to your site and make a call without being logged in (the SinchClient requires a user or token). For this tutorial, you will use a very basic backend for user management to get, started. You should not use this in a production environment. In production you should use your own user authentication. You can find a .net sample project here [http://www.sinch.com/tutorials/using-delegated-security-application-server-using-c-sinch-sdk/](https://github.com/sinch/net-backend-sample)

```html
<div class="container">
    <div id="auth" class="col-md-6">
        <h2>Login or register</h2>
        <form id="userForm">
            <input id="username" placeholder="username"><br>
            <input id="password" type="password" placeholder="password"><br>
            <button id="loginUser">Login</button>
            <button id="createUser">Create</button>
        </form>
    </div>
    <div id="call" class="col-md-6">
        <h2>Make a call</h2>
        <input type="text" id="phoneNumber" />
        <button id="callNumber">Call</button>
    </div>
    <div class="col-md-12" id="error"></div>
</div>
```

Next you need to create or login the user. 
1. Add a sinch client variable with your application key.
2. Hook up register button to a click event.
3. Hook up the login button to a click event.

```javascript
<script language="javascript">
$("#call").hide(); //hide the call ui
//1. Set up sinchClient
var sinchClient = new SinchClient({
    applicationKey: '<your application key>',
    capabilities: { calling: true },
    },
});
//2. Create user and start sinch for that user
$('button#createUser').on('click', function (event) {
    event.preventDefault();
    clearError();
    $('button#loginUser').attr('disabled', true);
    $('button#createUser').attr('disabled', true);
    var signUpObj = {};
    signUpObj.username = $('input#username').val();
    signUpObj.password = $('input#password').val();
    //Use Sinch SDK to create a new user
    sinchClient.newUser(signUpObj, function (ticket) {
        //On success, start the client
        sinchClient.start(ticket, function () {
            global_username = signUpObj.username;
            //On success, show the UI
        })
    })
});
//3. Login user and and start client
$('button#loginUser').on('click', function (event) {
    event.preventDefault();
    clearError();
    $('button#loginUser').attr('disabled', true);
    $('button#createUser').attr('disabled', true);
    var signInObj = {};
    signInObj.username = $('input#username').val();
    signInObj.password = $('input#password').val();

    //Use Sinch SDK to authenticate a user
    sinchClient.start(signInObj, function () {
        global_username = signInObj.username;
        //On success, show the UI
    })
});
</script>
```

## Handle errors
Its a good idea to take care of errors when you try and login to the client. To handle errors add a small error handler:
```javascript
var handleError = function(error) {
    //Enable buttons
    $('button#createUser').prop('disabled', false);
    $('button#loginUser').prop('disabled', false);
    //Show error
    $('#error').text(error.message);
    $('#error').show();
}
```
Next add a error `<div>` to the bottom on the page: 
```html
<div class="col-md-12" id="error"></div>
```
Add fail functionality sinchClient.newUser like below:
```javascript
 sinchClient.newUser(signUpObj, function(ticket) {
        //On success, start the client
        sinchClient.start(ticket, function() {
        global_username = signUpObj.username;
        //On success, show the UI
    }).fail(handleError);
}).fail(handleError);
```
Then, do the same for the login button: 
```javascript
//Use Sinch SDK to authenticate a user
sinchClient.start(signInObj, function() {
global_username = signInObj.username;
//On success, show the UI
}).fail(handleError);
```
Tidy up a bit by creating an clearError function and call it in your onClick handlers.
```javascript
    var clearError = function() {
        $('#error').text('');
        $('#error').show();
    }
```


## Add Calling UI
When the user has successfully logged in, you want to show a UI to make a call. Create a function to hide the login UI and show a calling UI:
```javascript
var showUI = function () {
        $('#auth').hide();
        $('#call').show();
    }
```
Add a call to showUI() by replacing the comments ```//On success, show the UI```

## Making a call
To make a call, you will need a couple more elements in your call div. Add the following into ```<div id="call">```.
```html
<button id="hangupCall" style="display: none">Hangup</button>
<audio id="incoming" autoplay></audio>
<audio id="ringtone" src="ringtone.wav" loop ></audio>
<div id="status"></div>
```
Next, add a click handler for the call button: 
```javascript
var callClient;
var call;
$('#callNumber').click(function (event) {
    event.preventDefault();
    callClient = sinchClient.getCallClient();
    $('#callNumber').attr('disabled', 'disabled');
    $('#phoneNumber').attr('disabled', 'disabled');
    $("#hangupCall").show();
    call = callClient.callPhoneNumber($('#phoneNumber').val());
    call.addEventListener(callListeners);
});
```
As you can see, you are missing event handlers for the call. A call can be in three states: *inprogress*, *established*, and *ended*; they occur in that order. What you want to do now is the following:

1.	When call is in progress, play a ringtone just like you hear on your regular phone.
2.	When the call is established (someone picked up or you are connected to their voicemail), you want to stop playing the ringtone and play sound from the call instead.
3.	When the call is ended, you want to shut down the audio from the call, and change the UI so the user can place a new call. In addition, your app will display some details about the call, like the duration.

```javascript
var callListeners = {
    //1. start playing sound and display status
    onCallProgressing: function (call) {
        $('#ringtone').prop("currentTime", 0);
        $('#ringtone').trigger("play");
        $('#status').append('<div id="stats">Ringing...</div>');
    },
    //2. Call is pickedup (or voicemail)
    onCallEstablished: function (call) {
        $('audio#incoming').attr('src', call.incomingStreamURL);
        $('audio#ringtone').trigger("pause");
        //Report call stats
        var callDetails = call.getDetails();
        $('#status').append('<div id="stats">Answered at: ' 
        + new Date(callDetails.establishedTime) + '</div>');
    },
    //3. Call ended, stop all sounds, and revert back
    onCallEnded: function (call) {
        /** you need to make sure the ringback is not 
        playing incase the call ended from progress (no answer)**/
        $('audio#ringtone').trigger("pause");
        $('audio#incoming').attr('src', '');
        $('#callNumber').removeAttr('disabled');
        $('input#phoneNumber').removeAttr('disabled');
        $("#hangupCall").hide();
        //Report call stats
        var callDetails = call.getDetails();
        $('#status').append('<div id="stats">Ended: '
        + new Date(callDetails.endedTime) + '</div>');
        $('#status').append('<div id="stats">Duration (s): ' 
        + callDetails.duration + '</div>');
        $('#status')
          .append('<div id="stats">End cause: ' + callDetails.endCause + '</div>');
        if (call.error) {
            $('div#callLog')
          .append('<div id="stats">Failure message: ' + call.error.message + '</div>');
        }
    }
};
```

## Hang Up button
Last but not least, add a click handler for the hang up button:
```javascript
$('#hangupCall').click(function (event) {
    event.preventDefault();
    call && call.hangup();
});
```

## Run it
Either you push this file to a webserver or run it locally. To give chrome access to your microphone you need to launch it with with the flag --allow-file-access-from-files
On Windows open a command prompt and type 
```pathtochrome%\chrome yourfilename.html --allow-file-access-from-files```
on Mac 
```open -a Google\Chrome yourfilename.html --allow-file-access-from-files```
Make sure chrome is completly shut down for it to work.
Complete source code can be found at [https://github.com/sinch/js-calling-tutorial](https://github.com/sinch/js-calling-tutorial)

## Summary
Now you have a good understanding of how the Sinch client works, and how to make a call with the Sinch client. The next steps will be to add your own authentication and make it more secure with the Sinch REST API.

Further reading:
- [Custom authentication with .net](http://tutorial.sinch.com/net-backend-sample)
- [Use the sinch JS SDK to build a messaging app](http://tutorial.sinch.com/js-messaging-tutorial/)

