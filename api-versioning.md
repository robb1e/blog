# API Versioning

Previously I wrote about how to avoid having the need for a different controller for each version of an API and how I believe that this practice is unnessasary and ultimately creating multiple resources where one should be sufficient. As part of that I touched upon how versioning could he handled and I had some comments about the style of versioning from some Tweets and from some colleges.

<blockquote class="twitter-tweet"><p>@<a href="https://twitter.com/moonmaster9000">moonmaster9000</a> @<a href="https://twitter.com/robb1e">robb1e</a> @<a href="https://twitter.com/jaderubick">jaderubick</a> you should version using a vendor string in the Accept header.</p>&mdash; David Celis (@davidcelis) <a href="https://twitter.com/davidcelis/status/335040861995933696">May 16, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

I wanted to dig a little deeper into this topic and look at some of the different ways people are using versioning for their resources on the web.

## Compatiblility as version control

Mark Nottingham has written a lot of the subject of API versioning and it's difficult to disagree with the high level arguments outlined in his [API evolution post](http://www.mnot.net/blog/2012/12/04/api-evolution). 

- keep compatible changes out of names
- avoid new major versions
- makes changes backwards-compatible
- think about forwards-compatibility

Nottingham adds that we [shouldn't use a custom header for versioning](http://www.mnot.net/blog/2012/07/11/header_versioning) and touches on versioing through the Accept header 

## Accept header

[RFC4288 section 3.2](http://tools.ietf.org/html/rfc4288#section-3.2) outlines how a vendor, i.e. an application, can make use of customisable MIME types in the Accept header. [Steve Klabnik](http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http#i_want_my_api_to_be_versioned) goes onto show how an application can extend this to include a version number as part of the customer header.

Looking at some concrete examples through the [well documented Github API shows how the Accept header can be used](http://developer.github.com/v3/media/) which ultimately look like:

`Accept: application/vnd.github[.version].param[+json]`

`curl -v -H 'Accept:application/vnd.example+json' localhost:3000`

### Parsing the Accept header

Extracting the version number would require some regular expression matching, something along the lines of: 

`"application/vnd.github.v2.full+json"[/^application\/vnd\.github\.v(\d)/, 1].to_i`

### Registering the MIME type

Alternatively you could register a mime type for each

in config/initializers/mime_types.rb

	Mime::Type.register "application/vnd.github.com.v1+json", :json_v1
	Mime::Type.register "application/vnd.github.com.v2+json", :json_v2
	
MIME types also allows parameters but would require registration for each one also:	

	Mime::Type.register "application/vnd.github.com+json; version=1", :json_v1
	Mime::Type.register "application/vnd.github.com+json; version=2", :json_v2
	
So that in your controller could look something like the following and deal with the request how you feel appropriate. 

	respond_to do |format|
		format.json_v1 { … }
		format.json_v1 { … }
	end
	
## Version via a request parameter

The thing with the version in an Accept header is when a GET request is sent to the server and I want to share that, I cannot send a URI and paste that into a web browser to see the same thing as my colleague. If the version is in the request parameter then we can send that around, put it in a browser and see the same thing. Just as seeing the version in the URI path makes it obvious what version number is beng used, seeing it in a request parameter also makes it obvious. 

## Version number as a date

The Foursquare API allows clients to send a version as a date in the format `yyyymmdd` which conviniently is an always increasing number. When a client starts to use the API they can put in that days date and the response will always be in that format. I like this concept as it removes the burden of having to know what endpoint to use and what version to send. The client mearly uses a known point in time and the response will always match if that is sent. If nothing is sent then the latest version of the resource is returned. 

With this in mind, I wrote a very small gem to put this into practice to demonstrate how I thought this could be achieved which boils down to the following. Given a number and a list of numbers, find the previous number in the list from the number given. For example, if the date `20130101` is passed and the list contains two dates of `20120101` and `20130601`, then `20130101` is returned.

	def find_version_for version, list
    	return list.last if version.nil?

	    selected_version = nil
    	list.each_cons(2) do |previous_version, next_version|
      	selected_version = next_version
      	if next_version > version
        	selected_version = previous_version
        	break
      	end
    	end

	    selected_version
  	end
  	
Using the returned number, the server can decide how to render the response to the client as defined in my previous post.  	


## Robustness principle

[Jon Postel](http://en.wikipedia.org/wiki/Jon_Postel) says "[Be conservative in what you do, be liberal in what you accept from others](http://en.wikipedia.org/wiki/Robustness_principle)", so why don't we allow our clients to do both?

