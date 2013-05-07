When the only consumer of an application data model are the views within the same application then the object design is fluid and maleable. Once an application exposes an API to more than one client, and especially if that client is on a different release cycle to the server, such as iPhone application, data models become rigid. Rails has always discouraged N-tier architecture to the benefit of development speed but APIs are contracts between a server and it's client and can be difficult to change. 

At first passing an object into the Rails JSON serialisation methods will work, but relying on the this will only get you so far. At some point a refactor will take place that will cause a breaking change. It could be something simple such as renaming a column, moving responsibilities from one class to another or adding extra meta-data to a response. Either way, adding this information into your model class starts to place more responsbilties into one place. 

There are a few ways out of this potential issue. Let's take another look at the `Post` object. The Rails rendering engine will call `as_json` on an object if the request has sent the `content-type` of `application\json` to the server.  Here we override the implementation in `ActiveRecord` to provide a stable, known version:

	def as_json(options={})
		{
			author_id: author.id
			title: title
		}
	end
	
A second option is to model the object explictly and serialise the internal model into a public representation. Although this can be seen as a step towards a N-tier architecture, it's also a means service dependant abstraction:

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
	
The benefit of doing this is a seperation of concerns. An application model doesn't need to know how it'll be represented by an API, command line interface or any other outside communication mechanism. You may lose some of the Rails `respond_with` goodness with this:

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
	
What's the difference between this and putting the `as_json` method into `Post` directly? More control, seperation of concerns (application modelling vs presentation) and the big win is when breaking changes occur within the API. Now we can put version relevant information into new objects, or into the serialised class itself.

	class Api::V2::Post
	  ...
	end
	
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
	    ...
	  end
	  
	  def v20121206
	    ...
	  end
	end

	
