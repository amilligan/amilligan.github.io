---
layout: post
title: Exceptions in controller specs
date: 2014-04-25
categories: Rails testing
---

When you’re testing a Rails controller, as when testing anything, you want to verify that your controller exhibits the desired behavior given certain conditions.  Rails controller tests make this somewhat difficult by not handling exceptions as the production controller infrastructure would.  You can rectify this situation with only a few lines of code.

For example, you would expect the controller to return a 404 response in response to a request for a nonexistent resource.

Consider this controller action:

```ruby
class WibblesController < ApplicationController
  def show
    @wibble = Wibble.find_by_param!(params[:id])
  end
end
```

A test, using Rspec, to verify this behavior might look something like this:

```ruby
describe "#show" do
  subject { -> { get :show, id: wibble } }

  context "when the requested wibble does not exist" do
    let(:wibble) { double(:wibble, to_param: "does-not-exist") }
    it { should respond_with_status(:missing) }
  end
end
```

Unfortunately, running this example results in the following failure:

```
||   1) WibblesController#show when the requested wibble does not exist should return 404
||      Failure/Error: subject { -> { get :show, id: wibble } }
||      ActiveRecord::RecordNotFound:
||        ActiveRecord::RecordNotFound
```

Our controller is doing exactly the right thing; when the application runs it will catch the `ActiveRecord::RecordNotFound` exception and return an appropriate 404 response.  However, the Rails test environment for controllers doesn't include this exception handling behavior, so the exception bubbles up to Rspec and Rspec reports the exception as a failure.

We can fix this failure in one of a few ways:

1. Change the controller behavior to not throw the exception, and return a 404 response.
2. Capture the exception, and return a 404 response.
3. Change the test environment to handle the exception more like a real environment.

Changing the controller action to not throw the exception is relatively easy; it could look something like this:

```ruby
class WibblesController < ApplicationController
  def show
    @wibble = Wibble.find_by_param(params[:id]) || head(404)
  end
end
```

Note that we replaced the `.find_by_param!` method with `.find_by_param`, so the finder will return `nil` if the record doesn't exist, and the statement will continue to the `head(404)` call.

You've probably noticed the first problem with this solution: we have to remember to add `|| head(404)` to every line of code that finds a record in a controller.  Clearly Rails must have a way to handle this without so much repetition.

This brings us to the second possible solution: using `.rescue_from` in the ApplicationController:

    class ApplicationController < ActionController::Base
      rescue_from ActiveRecord::RecordNotFound, with: :not_found

      def not_found
        head 404
      end
    end

Now no controllers need change how they find records, and our spec example passes.  However, we have one problem remaining: the `#head` method returns a status code, but with no response body.  This means that a user requesting a nonexistent resource will see a blank page, rather than a 404 page, in their web browser.  Here is a common solution that resolves this problem:

    class ApplicationController < ActionController::Base
      rescue_from ActiveRecord::RecordNotFound, with: :render_404

      def render_404
        render file: Rails.root.join("public", "404.html"), status: 404
      end
    end

Now our spec example passes, the controller will respond appropriately with a 404 response, and the user will see the 404.html provided in the public directory.  We could call this done, but we can still do better.

The default Rails behavior frequently returns a static 404 page from outside the Rails application file structure; Heroku, for example, expects you to provide static error pages in S3.  This means that exceptions which trigger the default 404 behavior (e.g. `ActionController::RoutingError`, which also results in a 404 response) may result in a different 404 page than `ActiveRecord::RecordNotFound`.  Does this matter?  Perhaps a malicious user could learn something about your system by analyzing which resources have routes and which do not.  At the least you may provide a slightly inconsistent user experience; if you think 404 pages don't matter, consider the work that GitHub put into [theirs](https://github.com/404).

More damningly, we've added a fair bit of code, and a static HTML page, that duplicate the default Rails behavior *just to make our tests pass*.  Tests should specify how code behaves, not affect the specifics of the implementation.  Nothing was wrong with our original implementation, so the test failures showed a problem in the tests, not the implementation.

This leads to the final solution to our original test failure: fix the test environment.  The real environment will catch and properly handle `ActiveRecord::RecordNotFound` exceptions, so the test environment should as well.  In the spec_helper.rb file:

    module HandleExceptionsInSpecs
      def process(*)
        super
      rescue ActiveRecord::RecordNotFound
        @response.status = 404
      rescue ActionController::UnknownFormat
        @response.status = 406
      end
    end

    RSpec.configure do |config|
      config.include(HandleExceptionsInSpecs, type: :controller)

      ...
    end

This simply decorates the `#process` method to handle exceptions and return the appropriate response status, as a real environment would.  No change to your controller implementation; no duplication of Rails-provided behavior; no inconsistencies in static HTML pages.

It should come as no surprise when developers change their code to make tests pass; properly written tests should regularly fail in meaningful ways.  But, remember that tests are simply code which can have flaws just as any code can.  When you come across a failure, always take a moment to consider whether the problem to be solved lies in the code or the tests themselves or even the testing infrastructure.

