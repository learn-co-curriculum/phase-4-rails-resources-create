# Rails Resource Routing: Create

## Learning Goals

- Use Rails to create a resource
- Understand the connection between the request body and `params`

## Introduction

In this lesson, we'll continue working on our Bird API by adding a `create`
action, so that clients can use our API to create new birds. To get set up, run:

```console
$ bundle install
$ rails db:migrate db:seed
```

This will download all the dependencies for our app and set up the database.

| HTTP Verb    | Path       | Controller#Action | Description            |
| ------------ | ---------- | ----------------- | ---------------------- |
| GET          | /birds     | birds#index       | Show all birds         |
| **POST**     | **/birds** | **birds#create**  | **Create a new bird**  |
| GET          | /birds/:id | birds#show        | Show a specific bird   |
| PATCH or PUT | /birds/:id | birds#update      | Update a specific bird |
| DELETE       | /birds/:id | birds#destroy     | Delete a specific bird |

<!-- ## Video Walkthrough -->
<!-- <iframe width="560" height="315" src="https://www.youtube.com/embed/wuzfkmOCe_U?rel=0&amp;showinfo=0" frameborder="0" allowfullscreen></iframe> -->

## Creating New Birds

As always, the first thing we'll need to do to add a new endpoint to our API is
update our routes. Following REST conventions, we'll want our clients to make a
POST request to `/birds` to create a new bird. Using the `resources` method, we
can create this route by adding in `create` to the list of actions we want
handled:

```rb
Rails.application.routes.draw do
  resources :birds, only: [:index, :show, :create]
end
```

After updating our routes, run `rails routes` to check what routes are now
available:

```txt
Prefix  Verb  URI Pattern           Controller#Action
 birds  GET   /birds(.:format)      birds#index
        POST  /birds(.:format)      birds#create
  bird  GET   /birds/:id(.:format)  birds#show
```

Awesome! We've successfully added a `POST /birds` route, which will run the
`create` in our `BirdsController`. Since we haven't set up that action, let's do
so now. For the time being, let's add in a `byebug` so that we can test out this
route and see what we'll need to do in order to create a new bird:

```rb
class BirdsController < ApplicationController

  # POST /birds
  def create
    byebug
  end

  # etc
end
```

Run your server now with `rails s`.

Now, we'll need to make a `POST /birds` with some data about the bird we're
trying to create. Recall from our schema that our `birds` table
has `name` and `species` columns:

```rb
create_table "birds", force: :cascade do |t|
  t.string "name"
  t.string "species"
  t.datetime "created_at", precision: 6, null: false
  t.datetime "updated_at", precision: 6, null: false
end
```

To create a new `Bird` instance, we'll need to provide values for these two
attributes. If we were making this request using `fetch`, it'd look like this:

```js
fetch("http://localhost:3000/birds", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
  },
  body: JSON.stringify({
    name: "Monk Parakeet",
    species: "Myiopsitta monachus",
  }),
});
```

Let's make that request using Postman (remember to add `Content-Type`:
`application/json` to the headers as well):

![birds post request with postman](https://curriculum-content.s3.amazonaws.com/phase-4/phase-4-rails-resources-create/birds-post-request.png)

After making the request, we'll hit our `byebug` debugger, so we can see all the
data about the request that we have access to:

```txt
    1: class BirdsController < ApplicationController
    2:
    3:   # POST /birds
    4:   def create
    5:     byebug
=>  6:   end
    7:
    8:   # GET /birds
    9:   def index
   10:     birds = Bird.all
```

From the `byebug` session, let's see how we can get access to the data we need
to handle this request. Remember, our goal in this action is to **create a new
bird** and **send the new bird object in the response**, so ultimately, we'll
want some code like this:

```rb
def create
  bird = Bird.create(name: ???, species: ???)
  render json: bird, status: :created
end
```

> The `status: :created` option will send a 201 status code, which indicates
> that the request has succeeded and has led to the creation of a resource.

To fill in the blanks, we'll need to figure out how to get data from the
**body** of the request, where our client sent the `name` and `species` for this
new bird.

In the `byebug` session, we can access the **entire** request object by using
the `request` method:

```txt
(byebug) request
#<ActionDispatch::Request POST "http://localhost:3000/birds" for ::1>
```

This `request` object has all kinds of info about what was sent in the request.
Try some of these methods out in your `byebug` session:

- `request.request_method`
- `request.headers["Content-Type"]`
- `request.body.read`

The last one, `request.body.read`, will read the body of the request as a
string. Nice! We could take it a step further, and parse that string as json:

```txt
(byebug) JSON.parse(request.body.read)
{"name"=>"Monk Parakeet", "species"=>"Myiopsitta monachus"}
```

This will return a Ruby hash of key-value pairs by parsing the JSON string from
the request body. However, that's a lot of steps for a fairly common task as a
Rails developer. Wouldn't it be nice if there was a bit of ✨ Rails magic ✨ to
make it easier to access that parsed request data? Enter the `params` hash!

## Using Params

We can more easily access all the information from the request body by using
`params`:

```txt
(byebug) params
#<ActionController::Parameters {"name"=>"Monk Parakeet", "species"=>"Myiopsitta monachus", "controller"=>"birds", "action"=>"create", "bird"=>{"name"=>"Monk Parakeet", "species"=>"Myiopsitta monachus"}} permitted: false>
```

We've seen `params` once before, as a way to access the dynamic part of the URL:

```rb
# GET /birds/:id
def show
  # params[:id] refers to the dynamic part of our route, defined by :id
  # a request to /birds/2 would give params[:id] a value of 2
  bird = Bird.find_by(id: params[:id])
  render json: bird
end
```

In this case, we can see that all the data from the body of our request has been
added to this `params` hash! Any time Rails receives a request with a
`Content-Type` of `application/json`, it will automatically load the request
body into the `params` hash. Let's use that information to create our bird. Exit
the `byebug` session by typing `continue` or `c` and hit enter. Then, update
your controller action like so:

```rb
def create
  bird = Bird.create(name: params[:name], species: params[:species])
  render json: bird, status: :created
end
```

Back in Postman, send the same request through. Now, you should see a response
in Postman with the newly created bird! You should also see in your Rails server
log the SQL that was executed when the bird was created:

```txt
Started POST "/birds" for ::1 at 2021-05-02 10:09:03 -0400
   (0.1ms)  SELECT sqlite_version(*)
Processing by BirdsController#create as */*
  Parameters: {"name"=>"Monk Parakeet", "species"=>"Myiopsitta monachus", "bird"=>{"name"=>"Monk Parakeet", "species"=>"Myiopsitta monachus"}}
  TRANSACTION (0.1ms)  begin transaction
  ↳ app/controllers/birds_controller.rb:5:in `create'
  Bird Create (2.0ms)  INSERT INTO "birds" ("name", "species", "created_at", "updated_at") VALUES (?, ?, ?, ?)  [["name", "Monk Parakeet"], ["species", "Myiopsitta monachus"], ["created_at", "2021-05-02 14:09:03.955909"], ["updated_at", "2021-05-02 14:09:03.955909"]]
  ↳ app/controllers/birds_controller.rb:5:in `create'
  TRANSACTION (0.9ms)  commit transaction
  ↳ app/controllers/birds_controller.rb:5:in `create'
Completed 201 Created in 15ms (Views: 0.5ms | ActiveRecord: 4.1ms | Allocations: 4408)
```

Success!

Experiment using Postman and `byebug`. What would you change if you wanted to
add additional keys to the `params` hash? What would you expect the server to
return if the new bird **wasn't** created successfully?

## Conclusion

We have now learned how to handle the `create` action. In the next lesson, we'll
explore the `params` hash further, and talk about ways to refactor our code
using additional features of the `params` hash.

## Check For Understanding

Before you move on, make sure you can answer the following questions:

1. When using `fetch` to make a `POST` request as opposed to a `GET` request,
   what additional property needs to be passed along with the `method` and
   `headers`?
2. How do we access this additional information to use it in our controller
   action?

## Resources

- [JSON Parameters](https://guides.rubyonrails.org/action_controller_overview.html#json-parameters)
