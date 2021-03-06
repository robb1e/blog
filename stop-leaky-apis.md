There are many blogs about how to expose an API for a Rails application and many times I look at this and am concerned about how these examples often leak the application design and the schema out through the API. When this leak occurs a change to the application internals can ripple out and break clients of an API, or force applications to namespace URI paths which I feel is unnecessary and ugly. 

When the only consumer of application data models are the views within the same application then the object design can be fluid and malleable. Once an application exposes an API to more than one client, and especially if that client is on a different release cycle to the server, such as iPhone application, data models become rigid. Rails tends discouraged N-tier architecture to the benefit of development speed but APIs are contracts between a server and it's client and can be difficult to change once they start being used. 

Passing an object into the Rails JSON serialisation methods will work for a time, but relying on this will only get you so far. At some point a refactor will take place that will cause a breaking change. It could be something simple such as renaming a column, moving responsibilities from one class to another or adding extra meta-data to a response. Either way, adding this information into your model class starts to place more responsibilities into one place. 

There are a few ways out of this potential issue. Let's take a look at the classic blog application and its `Post` object. The Rails rendering engine will call `as_json` on an object if the request has sent the `content-type` of `application\json` to the server.  Here we override the implementation from `ActiveRecord` to provide a stable, known version:

	def as_json(options={})
		{
			author_id: author.id
			title: title
		}
	end
	
A second option is to model the object explicitly and serialise the internal model into a public representation. We can duck-type the object to respond how `ActiveRecord` objects behave during a serialisation call. Although this can be seen as a step towards a N-tier architecture, it's also a step towards service dependent abstraction:

	class Api::Post
	  attr_reader :post
	  
	  def initialize(post)
	    @post = post
	  end
	  
	  def as_json(options={})
	    {
	      author_id: post.author.id
	      title: post.title
	    }
	  end
	end
	
The benefit of doing this is a separation of concerns between your data model and the data presentation. An application model doesn't need to know how it'll be represented by an API, command line interface or any other outside communication mechanism. If an application were tending more towards [HATEOAS](http://en.wikipedia.org/wiki/HATEOAS) for instance this separation could help resolve hyperlinks relevant to the interface. You may lose some of the Rails `respond_with` goodness with this:

	respond_to :html, :json
	
	def show
	  post = Post.find(params[:id])
	  respond_to |format| do
	  	format.html { @post = post }
	  	format.json { render json: Api::Post.new(post) }
	  end
	end
	

That can be regained with the help of a presenter:

	respond_to :html, :json

	def show
	  post = Post.find(params[:id])
	  @presenter = PostPresenter.new(post)
	  respond_with @presenter
	end
	
Where `PostPresenter` may look something like:

	class PostPresenter < SimpleDelegator
	  def as_json(options={})
	    Api::Post.new(self).as_json(options)
	  end
	end
	
What's the difference between this and putting the `as_json` method into `Post` directly? More control, separation of concerns with application modeling vs presentation and the big win is when breaking changes occur within the API. Now we can put version relevant information into new objects, or into the serialised class itself.
	
	class Api::Post
	  attr_reader :post, :version
	
	  def initialize(post, version)
	    @post = post
	    @version = version
	  end
	
	  def as_json(options={})
	  	send("v#{:version}")
	  end
	  
	  private
	  def v20130505
	    # version specific JSON
	  end
	  
	  def v20121206
	    # version specific JSON
	  end
	end
	
Through this we have versioning information in one place and through a request parameter of something like `v=20130506` the application can handle multiple versions in one object. For me, this ultimately removes URIs like `/v1/posts`, but why is that important? The URI is an identifier which points to a resource and having `v1` or `v2` in the URI muddies the fact that the two identifiers are pointing to the same resource. Using a request parameter, much like pagination is handled, means we can ask for a representation of that resource rather than having to specify different resources. Then we can do away with needing controllers such as `Api::V1::PostsController` and just deal with `Api::PostsController` or even just `PostsController` and deal with the versioning within the object instead of the URI path.


	
