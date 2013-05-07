# Queries should be done in the model only

Extending `ActiveRecord::Base` leaks a powerful API throughout an application which can lead to tempting code which breaks good design. Take the classic blog example where you may want to retrieve the latest posts by a given author.  You may have seen, or even written code that gets the dataset you need straight into the controller or view:

    Post.where(author_id: author_id).limit(20).order("created_at DESC").each { ... }
    
For me this is a design violation as well as breaking the ["Law of Demeter"](http://en.wikipedia.org/wiki/Law_of_Demeter). The example above tells me structure of the schema that the calling class has no business knowing. It also makes testing using stubs ugly and encourages testing against the database directly. A test would have to chain three methods to stub a return value. It's brittle, as in it's susceptible to breaking due to changes outside of the class.  For me it also fails from a narrative perspective in that it doesn't succinctly reveal the intent of this part of the application.

If we were testing this and attempting to use stubs, we'd have to write something like the below.  You can see how this is at best cumbersome, but also fragile.

    where = stub(:where)
    limit = stub(:limit)
    order = stub(:order)
    
    Post.stub(:where).with(author_id: author_id) { where }
    where.stub(:limit).with(20) { limit }
    limit.stub(:order).with("created_at DESC").and_yield(post1, post2, post3)
    
You may be forgiven for thinking you could chain the stubs like below, but the arguments are ignored and this just serves to highlight the breaking of the 'Law of Demeter'.

    Post.stub_chain(:where, :limit, :order).and_yield(post1, post2, post3)

I'd much rather see that as a message to the `Post` class.

    def self.latest_for_author id
      where(author_id: id).limit(20).order("created_at DESC")
    end
  
    Post.latest_for_author(1)
	
If there were variations of the limit and perhaps offset, they can be passed as option parameters of as an options hash:

	def self.latest_for_author id, limit = 20, offset = 0
	  where(author: id).limit(limit).offset(offset).order("created_at DESC")
	end
	
	Post.latest_for_author(1)
	Post.latest_for_author(1, 20, 0)
	
or

	def self.latest_for_author id, options
	  limit = options[:limit] || 20
	  offset = options[:offset] || 0
	  where(author: id).limit(limit).offset(offset).order("created_at DESC")
	end
	
	Post.latest_for_author(1, offset: 20)
	
In order to get the dataset the call looks like the following, and I think is more informative than using the ActiveRecord DSL directly.

    Post.latest_for_author(author_id).each { ... }
    
Testing is also easier, as it puts more emphasis on the messages being sent to objects rather than a chain of calls having to be correct.

    Post.should_receive(:latest_for_author).with(1).and_yield(post1, post2, post3)
    
There are a few advantages to this refactor:

- Only the `Post` class knows about the schema
- Any changes to the implementation of what `latest_for_author` are encapsulated in one place
- The method describes the intent more than the implementation
- Stubbing in the tests are easier as there is one clear dependency
- Testing the database is encouraged only in the class hitting the database

One further refactor could be done here, and that is to move the query logic out of the Post class once more, but this time into a purpose built query Object:

	class LatestPosts
	  attr_reader :author_id
	
	  def initialize author_id
	    @author_id = author_id
	  end
	  
	  def find_each(&block)
	    Post.where(author_id: author_id).limit(20).order("created_at DESC").find_each(&block)
	  end
	
	end
	
Where using the class looks like:	

    LatestPosts.new(author_id).find_each { ... }

Here's what [Bryan Helmkamp has to say on query objects](http://blog.codeclimate.com/blog/2012/10/17/7-ways-to-decompose-fat-activerecord-models/) in his excellent write up on fat ActiveRecord models. Bryan here rightfully points out that once in a single purpose object, they warrant little attention to unit testing. Now is the right time to use the database to ensure the right data set is being returned and that N+1 queries are not being performed. This means that database testing would only occur within the class actually hitting the database and not the rest of application which has a dependency on the database. 

All of these techniques discussed serve to improve the design of an application by preventing leaking responsibilities from one class throughout the rest of the application. I'm also not saying that developers shouldn't be using ActiveRecord or even Rails, but to use the tools responsibly.


