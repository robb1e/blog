# API Versioning

How to version an API has been a thoroughly discussed topic in the last several years regardless of protocol or approach, be that SOAP, REST or Hypermedia. Why contribute another post to the topic? My [last post on avoiding breaking changes through better design](http://pivotallabs.com/stop-leaky-apis) led to conversations in the office and online about versioning.

<blockquote class="twitter-tweet"><p>@<a href="https://twitter.com/moonmaster9000">moonmaster9000</a> @<a href="https://twitter.com/robb1e">robb1e</a> @<a href="https://twitter.com/jaderubick">jaderubick</a> you should version using a vendor string in the Accept header.</p>&mdash; David Celis (@davidcelis) <a href="https://twitter.com/davidcelis/status/335040861995933696">May 16, 2013</a></blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

I wanted to dig a little deeper into versioning and look at some of the different ways people are using versioning for their resources on the web.

## Compatiblility as version control

[Mark Nottingham](http://en.wikipedia.org/wiki/Mark_Nottingham) has written a lot of the subject of API versioning and it's difficult to disagree with the high level arguments outlined in his [API evolution post](http://www.mnot.net/blog/2012/12/04/api-evolution). 

- keep compatible changes out of names
- avoid new major versions
- makes changes backwards-compatible
- think about forwards-compatibility

In another post Nottingham adds that we [shouldn't use a custom header for versioning](http://www.mnot.net/blog/2012/07/11/header_versioning) and touches on versioning through the Accept header but there is a fundamental thread here: try as hard as possible to not introduce breaking changes so that versioning isn't a big issue.

One of the issues to content with when considering versioning is often building APIs requires more thought up front that an agile developer might be used to. URIs, Data structure, meta-data and extensibility are important and would be best considered up front. Once those decisions have been made changes to the structure often result in breaking older versions of the API. Anyone at an early stage of building an API would do well to put some thought into the API design and setting some rules for future consistency. 

## Accept header

The Twitter comment above pointed me towards using the Accept header as part of content negotiation and I found a number of blog posts covering the subject as well as some companies using this approach.

[RFC4288 section 3.2](http://tools.ietf.org/html/rfc4288#section-3.2) outlines how a vendor, i.e. an application, can make use of customisable MIME types in the Accept header. [Steve Klabnik](http://blog.steveklabnik.com/posts/2011-07-03-nobody-understands-rest-or-http#i_want_my_api_to_be_versioned) shows how an application can make use of this to include a version number as part of the Accept header used for content negotiation.

Looking at some concrete examples through the [well documented Github API](http://developer.github.com/v3/media/) shows how the Accept header can be used with their services:

`Accept: application/vnd.github[.version].param[+json]`

`curl -v -H 'Accept:application/vnd.example.v1+json' localhost:3000`

The `vnd` part is the vendor definition as outlined in [RFC4288](http://tools.ietf.org/html/rfc4288#section-3.2). Let's take a look at a number of options to make use of this header.

### Parsing the Accept header

Within an application, headers can be inspected from a HTTP request. Extracting the version number would require some regular expression matching, something along the lines of the following in an `application_controller` perhaps: 

    def api_version
	request.headers["Accept"][/^application\/vnd\.github\.v(\d)/, 1].to_i
    end

Once the application has this version number it can decide how to behave for the response.

### Registering the MIME type

An application could alternatively register a [MIME type](https://en.wikipedia.org/wiki/Internet_media_type) for each and use the `respond_to` blocks to decide how to render a response.

To register a MIME type in `config/initializers/mime_types.rb`

	Mime::Type.register "application/vnd.github.com.v1+json", :json_v1
	Mime::Type.register "application/vnd.github.com.v2+json", :json_v2
	
MIME types also allows parameters but would require registration for each one also:	

	Mime::Type.register "application/vnd.github.com+json; version=1", :json_v1
	Mime::Type.register "application/vnd.github.com+json; version=2", :json_v2
	
So a controller could look something like the following and deal with the request  appropriately. 

	posts = Api::Posts.all

	respond_to do |format|
		format.json_v1 { posts.v1.as_json }
		format.json_v1 { posts.v2.as_json }
	end
	
## Version via a request parameter

The thing with the version in an Accept header is the URI is difficult to share. If I wanted to share the URI with version information with a college I would have to send instructions on what `curl` arguments to send to the server to get the right response. It's can be as frustrating as trying to share a holiday on a website that renders pages based on what's in the session for the individual.  I would want to share a URI that someone can paste into a web browser address bar and see an appropriate response. Let's compare the two approaches:

`curl -v -H 'Accept:application/vnd.example.v1+json' localhost:3000`

vs

`curl -v localhost:3000/?version=v1`

or

`curl -v localhost:3000/?version=20130603`

## Version number as a date

The [Foursquare API](https://developer.foursquare.com) allows clients to send a version as a date in the format `yyyymmdd` which conveniently is an always increasing number. When a client starts to use the API they can use that days date and the response will always be in that format. I like this concept as it removes the burden of having to know what endpoint to use or what version to send in a header. The client mearly uses a known point in time and the response will always match if that is sent. If nothing is sent then the latest version of the resource is returned. 

With this in mind, I wrote a very small [gem](http://rubygems.org/gems/simple-versioner) to put this into practice to demonstrate how I thought this could be achieved which boils down to the following. Given a number (i.e. the vesion sent in the request) and a list of numbers (i.e. known versions with some change in an application), find the previous closest number in the list from the version number given. For example, if the date `20130101` is passed and the list contains two dates of `20120101` and `20130601`, then `20130101` is returned.

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

[Jon Postel](http://en.wikipedia.org/wiki/Jon_Postel) says "[Be conservative in what you do, be liberal in what you accept from others](http://en.wikipedia.org/wiki/Robustness_principle)", so why perhaps we could allow our clients to do both? 

Perhaps using the Accept header makes you a better denizen of the Internet, adhearing to [HATEAOS](http://en.wikipedia.org/wiki/HATEOAS) principles but I think using a version request parameter makes for a better Web experience. As a developer, I want to be able to put a URI into a browser and see a response rendered and for that I'd lean more towards the version parameter.

This approach can be generalised across other information sent from a client to a server. Clients like web browsers send information in every request, and these should honored as the defaults. The information sent includes what language, data and encoding formats the client would like. Using request parameters can offer overrides and support compatibility between different types of clients who want to use an API and a [cool](http://www.w3.org/Provider/Style/URI.html) [bookmarkable](http://www.w3.org/Provider/Style/Bookmarkable.html) URIs. 

