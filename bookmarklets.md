Building a bookmarklet provides an interesting challenge. It involves interaction on someone elses website where that site could be anything with any number of dependencies on CSS or Javascript libraries. The first choice to make is trying to work with that website and probably set an `!important` on every CSS selector used and hope that there's no namespace or versioning clashes with any Javascript included or use an iframe.

Iframes seem to have fallen out of favour in recent times but the sandboxed nature of the content inside an iframe mean the worries of CSS and Javascript clashes are gone. However this is replaced with a communications overhead of communicating between the host document and the iframe. I wanted to touch on some of the things our team did on a recent project to try and make this fairly seamless. 

## Getting into the DOM

A bookmarklet is a small piece of Javascript that a user can drag onto the bookmark bar and upon pressing the link the Javascript will run. Because there is no guarentee what site will be loaded and if that will have any number of Javascript libraries included it's best to use plain old Javascript to create an iframe element and append it to the body of the document. Our bookmarklet also needed to have some javascript if for nothing else but to be able to dismiss and remove the newly created elements. This can be done through the creation of a script element and appending to the body just like the iframe itself. 

Once the iframe and script tags are appended they are treated the same as any other element. The content is loaded and the script is executed. The next step is getting the window to talk to the iframe.  As a convienence the domain with protocol and port of the iframe is stored in a variable for later use.

	element=document.createElement('iframe');
    element.src='example.com?referrer=' + window.location;
    document.body.appendChild(element);

    script = document.createElement('script');
    script.src='example.com/bookmark.js';
    document.body.appendChild(script);

The location is also sent to the remote server to load the iframe so that it can also store that location for passing messages to the host.

## Sending a message

The [postmessage](https://developer.mozilla.org/en-US/docs/Web/API/window.postMessage) method is available to communicate bewtween the window and the iframe.  The host window with a reference to the iframe can call `postmessage` with a string as a message and as a security measure the target location. We had stored this during the loading of the elements as described above. 

	iframe.postmessage('hello', 'www.example.com');

That message won't get anywhere unless the iframe is listening for the `message` event on the other end.

	window.addEventListener('message', function(event){ … });
	
I've used the native Javascript above but really, once in the iframe itself the application has full control and could use JQuery or any other library at this point. Our application needs to listen to messages on both sides though so we needed the above to run in the host anyway.

## RPC

Sending a message is all well and good but any non-trival application is going to have more than one function to run. We took influence from [Remote Procedure Calls (RPC)](http://en.wikipedia.org/wiki/Remote_procedure_call) to call functions within the host and remote sites. The message sent was stringified JSON with a very light 'schema' of the function to run and parameters to send to the function.  

	{ 	
		f: 'theFunction',
		params: {
			...
		}
	}
	
The recipient could then parse the string it knew to be JSON, extract the function to call and call it with any optional parameters also sent. This does create a binding between the host and the iframe but as the appliction controlled both sides we deemed it an acceptable risk.  The function can be run from the `window` like so:

	window['theFunction']()
	
## Dealing with namespaces

It was mentioned earlier that one of the goals of using an iframe was not to clobber any Javascript namespaces but we did end up including Javascript in the host and to avoid this we used an application namespace. However calling that as a property on the `window` would no longer work. We looked to [ElementalJS](https://github.com/elementaljs) as an example of dealing with namespaced functions to parse the function.

	var fn = window; 
	var data = JSON.parse(event.data);
	var namespaced = data['f'].split('.');
	for (var i in namespaced) { 
		fn = fn[namespaced[i]];
	}
	fn(data.params);	
	 
This is natually fairly crude, some defensive code could be added but this demonstrates the intent of the processing. 

## Putting it all together

The host Javascript listens to the `message` event

	window.addEventListener('message', function(event){
		var fn = window; 
		var data = JSON.parse(event.data);
		var namespaced = data['f'].split('.');
		for (var i in namespaced) { 
			fn = fn[namespaced[i]];
		}
		fn(data.params);	
	});
	
When the iframe has loaded, it sends a ready message to the host

	$(document).on('ready', function(){
		var data = JSON.strinify({f: 'example.hello'});
		parent.postMessage(data, referer); // referer set by server from the request param for the iframe
	});
	
The host from the script loaded in the bookmarket has the `example.hello` function and it's run. This in turn replies to the iframe.

	var example.hello = function(){
		var iframe = document.getElementByTagName('iframe');
		var data = JSON.stringify({f: 'example.world'})
		iframe.contentWindow.postMessage(data, 'example.com');
	};
	
The iframe has an event listener which is the same code as the host, and runs the function `example.world`

	var example.world = function(){
		// hello, world
	};
	
## Wrapping up

This has shown some of the techniques for a 'hello, world' bookmark with two way communication between host and iframe that uses Javascript namespaces. 	