Most every Rails developer can successfully explain the relationship of HTTP methods and Rails controller actions: GET + singular path = #show, POST + plural path = #create, etc.[2]  Yet, nearly every Rails project I look at has controller actions that fall outside the standard "resourceful" routing.

I believe the reason is that thinking about a system in terms of only nouns (i.e. resources), with an extremely limited vocabulary of verbs, is very, very difficult.  As humans, we think in terms of actions:

1. I bake a pie
2. She slices the pie
3. You eat a slice of the pie

From an early age we learn that sentences include subject (I), verb (bake), and direct object (the pie).  How can we rewrite each of these sentences to use only CRUD verbs?  

* You **create** a pie by baking it, so we can use the **create** verb.  
  Resource: pies
  Action: create (since you **create** a pie by baking)
  Params: pie details (e.g. ingredients)
* Tweeting the pie doesn't actually affect the pie, but 
  Resource: tweets
  Action: create
  Params: reference to existing pie resource
* She slices the pie
  Resource: ???
  
Notice that the subject has little importance; we may care who performs an action for authorization purposes, but the primary elements of each are the direct object (the resource), and the action we perform on it.  Also, notice that while the pie is the direct object of each original sentence, it is only the target resource in the first action.


[2] If necessary, check out the [PeepCode REST cheat sheet](http://topfunky.com/clients/peepcode/REST-cheatsheet.pdf).
