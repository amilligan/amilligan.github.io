---
layout: post
title: Full error messages in JSON
date: 2014-04-25
categories: ruby JSON
---

Rails has made it easy to generate JSON responses to requests, which should make life easier for any client developer (iOS, Android, Backbone, what have you).  Fortunately, this includes error responses, which Rails will generate automatically (with a 422 response code) when model validation fails.

The default JSON for a model which has failed validation looks generally like this:

```json
{
  "errors": {
    "name": ["may not be silly"]
  }
}
```

Unfortunately, turning that collection of errors into messages for display to the user can be a pain in languages that don't have the inflection support of Rails.  This format also limits the messages that the server can send, since the name of the field must be the first word in the error message.

This JSON comes from the `ActiveModel::Errors#to_json` method, which provides the `full_messages` option for turning the error message fragments into full sentences.  Unfortunately, the JSON builder interface doesn't expose this option, so to use it you have to do a small bit of work.  Simply include the following in an initializer:

```ruby
module FullMessageErrors
  def as_json(options = {})
    super(options.merge(full_messages: true))
  end
end
ActiveModel::Errors.instance_eval { prepend FullMessageErrors }
```

Now you have error messages that clients can easily consume:

```json
{
  "errors": {
    "name": ["Please select an appropriately serious display name"]
  }
}
```

Finally, since this uses the built-in string manipulation in Rails, the generated error messages should respect any localization files you have in place.

