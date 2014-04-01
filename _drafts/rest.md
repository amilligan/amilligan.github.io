These days you can’t turn around twice quickly without bumping into a web service that describes itself as RESTful.   At the same time, each supposedly RESTful web service seems to have more problems and deviations from the definition of REST than the last.  Dr. Fielding wrote about this problem[1] years ago, and little seems to have improved since.  From the comments on his post:

> *Roy T. Fielding says:*
> I think most people just make the mistake that it should be simple to design simple things. In reality, the effort required to design something is inversely proportional to the simplicity of the result. As architectural styles go, REST is very simple.

Dr. Fielding wrote that in response to the question of why developers so frequently get REST wrong.  I won't claim to have a complete conceptual understanding of REST -- I can't say I have read and understood his entire dissertation[2] -- but I feel I have begun to appreciate the beauty of its simplicity.  In the interest of offering some insight for someone new to REST, or perhaps getting some feedback from someone with an understanding that differs from mine, here are the basic principles that I follow when writing RESTful interfaces in Rails:

#### REST ain't APIs

Nearly every reference I see to REST refers to an API.  REST isn't a structure for building APIs, but a structure for building web services.  RESTful web services may return different representations of resources; the most common representations are HTML, JSON, and perhaps XML.

This means that *your API is not RESTful*; your web service may be RESTful, and may return a machine-parsable (e.g. JSON, XML, etc.) representation of resources, commonly referred to as an API.

#### A resource is not a database record

From [2]:
> Any information that can be named can be a resource: a document or image, a temporal service (e.g. "today's weather in Los Angeles"), a collection of other resources, a non-virtual object (e.g. a person), and so on. In other words, any concept that might be the target of an author's hypertext reference must fit within the definition of a resource.

Consider a web application in which Alice sends an invitation to Bob.  Bob may accept or ignore the invitation; either action will modify the record created when Alice sent the invitation, as well as potentially invoke external actions (e.g. send Bob an email, provision resources for Bob, etc.).  You could perhaps implement this as an `update` action on the `invitation` resource (HTTP PATCH), changing the value of the `accepted` attribute.  But, the HTTP standard states that PATCH requests must be idempotent; 

#### Every action operates on exactly one resource[3].

From the referenced article[1] (emphasis mine):

> A REST API should never have “typed” resources that are significant to the client. Specification authors may use resource types for describing server implementation behind the interface, **but those types must be irrelevant and invisible to the client**.  The only types that are significant to a client are the current representation’s media type and standardized relation names.

I most often see this violated in Rails controllers that provide the standard resourceful actions, and then one or two extra actions that don't quite fit into the available actions:

```ruby
class InvitationsController < ApplicationController
  # POST /users/alice/invitations
  def create
    @invitation = @user.invitations.create(params[:invitations])
    respond_with(@invitation)
  end

  # POST /users/alice/invitations/bulk_invite  ???
  def bulk_invite
    @invitations = params[:invitee_ids].collect do |invitee_id|
      @user.invitations.create(params[:invitation].merge(invitation_id: invitation_id))
    end
    respond_with(@invitations)
  end
end
```

What to do if one of the invitee IDs doesn't refer to a valid user?  How can we record when a user has sent multiple invitations, and to whom?  How can we check the current user's authorization to send individual invitations vs. send bulk invitations?  How does the client find the non-standard bulk_invite URL?

Adding a distinct BulkInvitations Rails controller with a standard `#create` action changes the request from a standard action that creates and returns one `bulk_invitation`.  Creating a single `bulk_invitation` may result in the creation of multiple `invitation` records in the underlying data model; the server's underlying data model (i.e. database schema) need not map directly to the web service's resource model[4].

#### Every resource has a unique location.

I most often see this violated in one of two ways in Rails:

1. The JSON (or XML) representation of a resource includes an `id` attribute, but no `location` (or `url`) attribute.  Without a canonical location (or URL) for the resource, the assumption is that the client of the interface will construct the location by appending the ID attribute to a location fragment provided by the parent resource.  In different contexts this can lead to multiple URLs for the same resource.

    <<<<<<<<<<<<<<<<<<<<<<<<<<<<<EXAMPLE

    Unfortunately, many client libraries like Backbone codify this behavior.  Fortunately, modifying JavaScript libraries to behave appropriately is fairly straightforward.

2. Content-specific resource locations.  I commonly see web services that have URLs like `/users/alice/invitations` for HTML content, and `/api/v1/users/alice/invitations` for JSON content.  Alice has only one collection of invitations, and therefore only one invitations collection resource.  Each RESTful resource has one unique location, entirely independent of the content of the returned representation[5], so these separate locations violate REST.

#### Each resource should specify the location of its available subresources.

Nearly every web service I look at gets this wrong.  From the referenced article[1]:

> A REST API must not define fixed resource names or hierarchies (an obvious coupling of client and server). Servers must have the freedom to control their own namespace. ... [Failure here implies that clients are assuming a resource structure due to out-of band information, such as a domain-specific standard, which is the data-oriented equivalent to RPC's functional coupling]

In words that people like me can understand: the client should make no assumptions about the location of resources.

Consider the following JSON, from `/users/alice`

```json
{ "name": "Alice" }
```

How would you fetch a list of Alice's messages?  You could look at the API documentation, find that users have a `messages` subresource, and construct the location for the request as `/users/alice` + `/messages`.  You've now tightly coupled your client implementation to a fixed resource hierarchy; if the server changes the location of the messages resource (perhaps simply to `/messages`) your client will break.

Now consider this JSON:

```json
{ 
  "resources": {
    "messages": { "location": "/users/alice/messages" }
  }
  , name: "Alice"
}
```

The client now need make no assumptions about where to send the request.  Additionally, the server can simply not include resources that the current user does not have authorization to access, and additional authorization information for resources for which the client has limited access.  For example:

```json
{ 
  "resources": {
    "messages": {
      "location": "/users/alice/messages", "authorizations": ["create"]
    }
  }
  , name: "Alice"
}
```

#### A resource is the same, no matter the representation format.

I see this violated most often in Rails controller that return more than one format in a single action, or (worse) two different controllers that each return a distinct representation of a single resource.  For example:

```ruby
class MessagesController < ApplicationController
  # GET /users/alice/messages
  def index
    respond_to do |format|
      format.html { @messages = @user.unread_messages.limit(10) }
      format.json { render json: @user.messages }
    end
  end
end
```

This action clearly returns different data based on the desired representation of the resource.  Each response is simply a representation of the resource and therefore should provide an accurate view of the resource, regardless of format.

If a web service needs to return different data (messages vs. unread messages), it should provide distinct resources for each.

[1] http://roy.gbiv.com/untangled/2008/rest-apis-must-be-hypertext-driven

[2] https://www.ics.uci.edu/\~fielding/pubs/dissertation/top.htm

[3] I consider a plural path to refer to a single collection resource.  In other words, fetching `/users/alice/pie-recipes` returns representation of the collection resource, not simply a collection of representations of individual resources.  I believe this conforms to Dr. Fielding's intentions.

[4] Mechanisms for transitioning between an application's data model and the resource model for that application's web service is a separate, and potentially length, discussion.  For the purpose of (relative) brevity I've tried to restrict the scope of this article to ActionController's responsibilities.

[5] But, what about API versioning?  A lot of people have written about versioning APIs with HTTP headers, rather than with distinct locations; I leave researching those articles as an exercise for the reader.

