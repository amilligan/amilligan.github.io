Rails has, since its creation, contained a lie.  This lie contains enough truth that it has gone uncorrected, and largely unnoticed, while simultaneously causing the occasional headache.  You see the lie every time you run `rake routes`:

```bash
% rake routes
Prefix Verb URI Pattern            Controller#Action
wibble GET /wibbles/:id(.:format) wibbles#show
```

The lie that you see is that `:id` in the URL definition.  This lie leads to a lot of controller actions that look like this:

```ruby
class WibblesController < ApplicationController
  def show
    @wibble = Wibble.find_by_id!(params[:id]) # [1]
  end
end
```

The lie is, of course, that the `:id` parameter is *not* the ID[2] of the desired record; it is the *URL parameter* of the desired record.  If you find this difficult to believe, check out how Rails's own URL helpers generate URLS.  You'll find that this:

```ruby
wibble_path(@wibble) # => "/wibbles/1"
```

does not use the `#id` attribute of the `@wibble` object to build the URL, but instead uses the result of calling the `#to_param` method[3]. 

ActiveRecord has perpetuated the lie with its default implementation of the `#to_param` method[4]:

```ruby
def to_param
  id && id.to_s # Be sure to stringify the id for routes
end
```

The two obvious questions are:

1. Why do I care?
2. Assuming I care, what should I do about it?

You can get away without caring, of course; after all, the vast majority of Rails projects do not make the distinction.  However, if you ever choose to change the URLs for your resources (e.g. `/users/sally` rather than `/users/1234`) you'll have to change the finder method you use in your controllers.  Why should a controller be coupled to the mechanism by which a resource is found or instantiated?

Not all common resources use a database ID as their URL parameter by default.  Consider the [password reset mechanism](http://rubydoc.info/github/plataformatec/devise/master/Devise/Models/Recoverable) in [Devise](https://github.com/plataformatec/devise): the URL parameter for the password reset is a GUID-style token, not a database ID.  

More importantly, the data model of an application should not define the resource model of that application's web interface.  From section 5.2.1.1 of Dr. Roy Fielding's dissertation[5]:

> The key abstraction of information in REST is a resource. Any information that can be named can be a resource: a document or image, a temporal service (e.g. "today's weather in Los Angeles"), a collection of other resources, a non-virtual object (e.g. a person), and so on. In other words, any concept that might be the target of an author's hypertext reference must fit within the definition of a resource. A resource is a conceptual mapping to a set of entities, not the entity that corresponds to the mapping at any particular point in time.

Imagine your application manages virtual machines in a managed cloud system[6]; URLs for specific machines may very well not contain a database ID. Again, why should your controllers care?

Assuming you decide to care, you can encapsulate the retrieval mechanism for your data models from your controllers fairly easily; ActiveRecord has already provided half of the abstraction you need with `#to_param`.  You can use a gem such as [find_by_param](https://github.com/bumi/find_by_param), which provides the abstract finder function as well as helpers for generating pretty URLs.  You can also get the very basic functionality by adding a short initializer to your project:

```ruby
class ActiveRecord::Base
  class << self
    def find_by_param(param)
      find_by_id(param)
    end

    def find_by_param!(param)
      find_by_id!(param)
    end
  end
end
```

These default `#find_by_param(!)` methods behave as the proper inverse of the default `#to_param` method provided by ActiveRecord.  If you use them to retrieve models in controller actions, then you can change the URL representation or retrieval mechanism for any model class without affecting your controllers.

[1] Or, worst of all, with the fantastically overloaded `#find`:

```ruby
@wibble = Wibble.find(params[:id])
```

[2] It is a form of identifier for the record, but not the one that you would expect to receive if querying the record for its `#id` property.

[3] Incidentally, Rails controller specs also call `#to_param` on objects passed as parameters to test requests:

```ruby
  describe "#show" do
    subject { -> { get :show, id: wibble } } # calls #to_param on wibble
    it { should respond_with_status(:ok) }
  end
```

[4] I haven't included the code comments here, but they are [worth reading](http://api.rubyonrails.org/classes/ActiveRecord/Integration.html#method-i-to_param).

[5] https://www.ics.uci.edu/\~fielding/pubs/dissertation/rest_arch_style.htm#sec_5_2

[6] Very much like described [here](http://www.tbray.org/ongoing/When/200x/2009/03/20/Rest-Casuistry)

