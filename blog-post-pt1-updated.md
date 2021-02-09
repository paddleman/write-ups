# User Registration, Authentication, and Authorization with Ember Octane and Phoenix

Part 1: Phoenix Setup and User Registration

December 19, 2020.

**Fair warning**:  *I am a hack. I am an civil engineer that has a hobby doing web development. Follow me at your own risk!* That said, others may find it worthwhile to read my experiences in developing a web app.

Big Thanks to [**Embercasts**]() excellent course on using [Ember with a Phoenix backend]().  I have watched it three times, and bits and pieces over and over again. I cannot stress enough how well presented and thourough Ryan Tablada and Erik Brin were in putting this course together. The course is a couple of years old; it uses Ember v3.0.X, --which still works in the current version. It is still well worth the purchase price. The last time I went thorough the videos was to update the Ember code to Octane -- which turned out to be an excellent exercise in self-learning to understand Octane syntax, patterns, and implementation.

That said, I have made some changes to packages used, obviously updated to the current versions of both Ember and Phoenix, and implemented a few thing is different ways -- but full credit to previous works.

For this application, I will be using [Elixir Phoenix](https://www.phoenixframework.org/) (v1.5.5) as the backend that will serve up [JSON:API](https://jsonapi.org/) complient json to be used by an frontend developed in [Ember.js](https://emberjs.com/) ---  specifically *[Ember Octane](https://blog.emberjs.com/2019/12/20/octane-is-here.html)* --- this app will use Ember(v3.23).

This write up assumes you have all the necessary precursors installed as I will not be going over any of that. Suffice to say that you have to have elixir, mix, phoenix etc installed, posgresql (and postgis -- *just because* -- ) as well as npm, yarn, ember-cli etc, etc. I will try to point out where I found that additional programs or dependencies needed to be updated or installed, but realize YMMV.

A note about the application. It is called **pucks** --- like hockey pucks.  I am Canadian. I like hockey. If this series of blog posts evolves into an on-going thing, then most folks will be able to understand the basic concepts and structure of players, leagues, teams, seasons, etc.

Server Side.

Let's create a new phoenix application *sans* the html and webpack stuff since the application is a backend api and not a server-side application. We then cd into the newly created directory.

```
mix phx.new pucks --no-html --no-webpack
cd pucks
code .
```

This gets us to the basic template for a phoenix app that is intended as a backend api.  Now let's customize it to our liking.

First off, open up the mix.exs file and add in our dependencies.  I am going to be adding in all we need up front --  explanations will follow later as we start using them. Most of the code I add comes straight from the docs of individual packages, or in some cases endless internet searches looking for answers to why things ain't working.

So... dependencies.

```elixir
# mix.exs
# ...
defp deps do
[
# ...

  {:ja_serializer, github: "vt-elixir/ja_serializer", branch: "master"},
  {:cors_plug, "~> 2.0.2"},
  {:geo_postgis, "~> 3.3.1"},
  {:ueberauth, "~> 0.6.3"},
  {:ueberauth_identity, "0.3.1"},
  {:guardian, "2.1.1"},    
  {:comeonin, "~> 5.3.1"},
  {:bcrypt_elixir, "~> 2.2.0"}
]
end

```
We added ja_serializer to handle the serialization into JSON:API compliant JSON. I pulled it directly from github because, as of this writing, there were a few pull requests that were in master that were not in the latest release.

[cors_plug](https://github.com/mschae/cors_plug) takes care of CORS stuff.

[geo_postgis](https://github.com/bryanjos/geo_postgis) because I do a lot of GIS in my day to day, have been using GIS since school in 1991, and I have aspirations about having a spatial component to this application one day. [Postgis](https://postgis.net/) is the fantastic spatial extentions to Postgresql. [geo](https://github.com/bryanjos/geo) and geo_postgis are elixir wrappers that allows you to declare spatial types (that map to posgis types) and call certain postgis functions from elixir. We won't be using them off hop -- ignore it if you want, but configuring the database and such will assume it is installed and configured.

I will use [Json Web Tokens](https://jwt.io/) (JWT) for authenication to the frontend client.  The server will be using the [ueberauth](https://github.com/ueberauth/ueberauth) for user authenication, and the [ueberauth_identity](https://github.com/ueberauth/ueberauth_identity) "strategy" -- which is basic user validation using <i>username</i> and <i>password</i>.

[Guardian](https://github.com/ueberauth/guardian) will provide the JWT token functionality -- encoding/decoding and signing/verifying.

[comeonin](https://github.com/riverrun/comeonin) and [bcrypt_elixir](https://github.com/riverrun/bcrypt_elixir) will take care of password hashing and verification.

Okay, let's actually add the packages to the application -- you know the drill.

```
mix deps.get
```
Now for the configuration, starting with ja_serializer.

```elixir
# config/config.exs

# ...

# Use Jason for JSON parsing in Phoenix
config :phoenix, :json_library, Jason

#ja_serializer setup
config :phoenix, :format_encoders,
  "json-api": Jason

config :mime, :types, %{
  "application/vnd.api+json" => ["json-api"]
}

# geo_postgis to use Jason
config :geo_postgis,
json_library: Jason

# ...

```

We will use the Jason elixir library for JSON. Feel free to use Poison if you want, but since Phoenix adopted Jason, why blow against the wind?

Same with geo_postgis which uses Poison by default.

Configuration for the authorizaiton libraries will a couple of steps from now.


Following the ja_serializer docs... clean and build the mime type.

```
mix deps.clean mime --build
```
.... and before I forget, let's accept json-api and add the necesarry plugs to our pipeline in the router.

```elixir
# ./lib/pucks_web/router.ex
# ...

  pipeline :api do
    plug :accepts, ["json", "json-api"]
    plug JaSerializer.ContentTypeNegotiation
    plug JaSerializer.Deserializer  

  end

  scope "/api", PucksWeb do
    pipe_through :api
  end

# ...
```
Some geo_postgis housekeeping: we need to add the Posgres types to the application. Create a file under ```lib``` called ```postgres_types.ex```. It will be referenced in our ecto configuation.

```elixir
# ./lib/postgres_types.ex
Postgrex.Types.define(Pucks.PostgresTypes,
              [Geo.PostGIS.Extension] ++ Ecto.Adapters.Postgres.extensions(),
              json: Jason)

```
As promised, let's use the types in our database configuration:

```elixir
#./config/dev.exs
# ...

# Configure your database
config :pucks, Pucks.Repo,
  username: "postgres",
  password: "postgres",
  database: "pucks_dev",
  hostname: "localhost",
  show_sensitive_data_on_connection_error: true,
  pool_size: 10,
  adapter: Ecto.Adapters.Postgres,
  types: Pucks.PostgresTypes

# ...
```
 Alright then. Let's try to create the database.

 ```mix ecto.create```

Success!  At this point, I open up pgAdmin to see the newly minted database (pucks_dev), and I add the postgis extension to said database. This is as easy as right-clicking on the database, selecting **CREATE EXTENSION** and then scrolling through to the postgis extension and clicking *Save*. You will see the posgis extension added in under *Extensions*.

Almost forgot. We have to configure CorsPlug. Again, following the docs, and opting for the recommended "put it in the endpoint" ... just ahead of the *Router* plug:

```elixir
#./lib/pucks_web/endpoint.ex
# ...

  plug CORSPlug, origin: ["http://localhost:4200", "http://example2.com"]

  plug PucksWeb.Router
end
```
I am accepting requests from localhost:4200 since that is the default port for Ember applications.  I guess I also except requests from *example2.com* but that is only because it will serve as a "how-to" reminder should I want to add more origins in the future.

Just for fun, let's serve the application.

```iex -S mix phx.server```

Okay... it's running. A reminder **ctrl-c** twice stops the server.

#### Editing the phx generators

Before we go any further, I want change the controller template that is used when I run the ```mix phx.gen.json``` generator.  There is a pattern for controllers (and views for that matter) when using *ja_serializer* and the JSON:API spec for Phoenix applications. Rather then editing the generated controller file (or creating it new from scratch) each time, we will edit the template so the generator does it for us, so to speak.

How do you do this? Easy. Under the ```./priv``` directory, create a structure like so: ```./priv/templates/phx.gen.json```. In that directory, create a ```controller.ex``` file.  

Here is the controller template that I use for a standard CRUD model:

```elixir 
# ./priv/templates/phx.gen.json/controller.ex

defmodule <%= inspect context.web_module %>.<%= inspect Module.concat(schema.web_namespace, schema.alias) %>Controller do
  use <%= inspect context.web_module %>, :controller

  alias <%= inspect context.module %>
  alias <%= inspect schema.module %>

  action_fallback <%= inspect context.web_module %>.FallbackController

  def index(conn, _params) do
    <%= schema.plural %> = <%= inspect context.alias %>.list_<%= schema.plural %>()
    render(conn, "index.json-api", data: <%= schema.plural %>)
  end

  def create(conn, %{"data" => data = %{"type" => <%= inspect schema.plural %>, "attributes" => <%= schema.singular %>_params}}) do
    with {:ok, %<%= inspect schema.alias %>{} = <%= schema.singular %>} <- <%= inspect context.alias %>.create_<%= schema.singular %>(<%= schema.singular %>_params) do
      conn
      |> put_status(:created)
      |> put_resp_header("location", Routes.<%= schema.route_helper %>_path(conn, :show, <%= schema.singular %>))
      |> render("show.json-api", data: <%= schema.singular %>)
    end
  end

  def show(conn, %{"id" => id}) do
    <%= schema.singular %> = <%= inspect context.alias %>.get_<%= schema.singular %>!(id)
    render(conn, "show.json-api", data: <%= schema.singular %>)
  end

  def update(conn, %{"data" => data =  %{"id" => id, "type" => <%= inspect schema.plural %>, "attributes"=> <%= schema.singular %>_params}}) do
    <%= schema.singular %> = <%= inspect context.alias %>.get_<%= schema.singular %>!(id)

    with {:ok, %<%= inspect schema.alias %>{} = <%= schema.singular %>} <- <%= inspect context.alias %>.update_<%= schema.singular %>(<%= schema.singular %>, <%= schema.singular %>_params) do
      render(conn, "show.json-api", data: <%= schema.singular %>)
    end
  end

  def delete(conn, %{"id" => id}) do
    <%= schema.singular %> = <%= inspect context.alias %>.get_<%= schema.singular %>!(id)

    with {:ok, %<%= inspect schema.alias %>{}} <- <%= inspect context.alias %>.delete_<%= schema.singular %>(<%= schema.singular %>) do
      send_resp(conn, :no_content, "")
    end
  end
end
```

I included this file because (a) I do this when I create a Phoenix server conforming to JSON:API specs, and (2) I actually figured out how to edit the phx generators and someone else might find this tidbit useful.

For our User model, the reality is this file will be chopped down since we will not want all these methods --- really only ```:create```, ```:delete```, a custom ```:show_current``` which will authenciate a user -- and I will keep a ```:index``` method so I can confirm and see the users I created.  

Before we add the User functionality, let's commit our work thus far.
```
git init
git add .
git commit -am 'initial commit with ja_serializer, cors_plug, geo_postgis configured'
```
## Adding a User to the application. 

We need users to authenticate so lets add that functionality.

Okay, we will run a generator to create a *user*. I will put it in a **People** context.

``` mix phx.gen.json Accounts User users username:string email:string password:string password_confirmation:string password_hash:string```
 
We included a boat load of password fields in the command, but we will be editing our migration and schema to get them to where they have to be.

First the migration file (```./priv/migrations/{timestamp}_create_users.exs```)-- delete the ```password``` and ```password_confirmation``` fields. These should not be persisted in the database, since that is purpoes of hashing the password.  These fields will be tagged as *:virtual* fields in the schema.  

We can execute the migration to create the table and fields in the database.

```mix ecto.migrate```

We can also add the user resouce to the router.

```elixir
# ./pucks_web/router.ex
#  ...

  scope "/api", PucksWeb do
    pipe_through :api

    resources "/users", UserController, only: [:create, :index, :delete] 
  end

#  ...
```
Notice we are limiting the methods available for the ```/users``` endpoint

Now for the schema. We put the *User* schema in the **People** context through our generator.

Open the schema file ```./lib/pucks/people/user.ex``` .

Change the ```password``` and ```password_confirmation``` to *:virtual* fields -- meaning they will not be persisted in the database.
```elixir
# ./lib/pucks/people/user.ex
#  ...
    field :password, :string, virtual: true
    field :password_confirmation, :string, virtual: true

    timestamps()
  end
#    .....
```

Remove password_hash from the cast and validation function, and while we are at it we can add a few more validations: 
```elixir
# ./lib/pucks/people/user.ex
#  ...

  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:username, :email, :password, :password_confirmation])
    |> validate_required([:username, :email, :password, :password_confirmation])
    |> unsafe_validate_unique([:username], Pucks.Repo)
    |> unsafe_validate_unique([:email], Pucks.Repo)
    |> validate_confirmation(:password)    

  end
end
```
The *unsafe_validate_unique* validation included with **Ecto.Changeset**, as the name suggests, checks for uniqueness of a field (*email* and *username* in our case). In order to do so, we also have to pass in the **Repo** for the method to check against as the second argument.

The only thing left to do is to hash the password. This is where *bcrypt_elixir* comes in.

We create a new private method that will hash_the_password, using the ```add_hash/2``` function provided by Bcrypt_elixir. We wrapped this in a case statement to test is the changeset is valid. 


```elixir
# ./lib/pucks/people/user.ex
#  ...
  defp hash_the_password(changeset) do
    case changeset do
      %Ecto.Changeset{valid?: true, changes: %{password: password}} ->
        hash = Bcrypt.add_hash(password)

        put_change(changeset, :password_hash, hash.password_hash)
      _ ->
        changeset       
    end
  
  end
#   ...
```
And finally we add it as the final pipe to our changeset.

```elixir
# ./lib/pucks/people/user.ex
#  ...

  |> hash_the_password()

#   ...
```
At this stage, probably not a bad idea to fire up the phx server and test it out.

```zsh
> iex -S mix phx.server
```
```zsh
iex(1)> alias Pucks.People.User
Pucks.People.User
iex(2)> attrs = %{ email: "bobby.clarke@flyers.com", username: "bobby", password: "number16", password_confirmation: "number16"} 
%{
  email: "bobby.clarke@flyers.com",
  password: "number16",
  password_confirmation: "number16",
  username: "bobby"
}
iex(3)> user = User.changeset(%User{}, attrs)

[debug] QUERY OK source="users" db=0.3ms decode=0.8ms queue=0.8ms idle=503.5ms
SELECT TRUE FROM "users" AS u0 WHERE (u0."username" = $1) LIMIT 1 ["bobby"]
[debug] QUERY OK source="users" db=0.3ms queue=0.8ms idle=509.6ms
SELECT TRUE FROM "users" AS u0 WHERE (u0."email" = $1) LIMIT 1 ["bobby.clarke@flyers.com"]
#Ecto.Changeset<
  action: nil,
  changes: %{
    email: "bobby.clarke@flyers.com",
    password: "number16",
    password_confirmation: "number16",
    password_hash: "$2b$12$yeeAXHE9gVOtvEFiJs85auzwBfjPCo/EJQ/8EDqZnbUTKR3HPhm/y",
    username: "bobby"
  },
  errors: [],
  data: #Pucks.People.User<>,
  valid?: true
>
```
We now have a vaild changeset. We can insert the record (stored in the variable <i>user</i>) into our database.

```zsh
iex(4)> Pucks.Repo.insert user

[debug] QUERY OK db=1315.5ms queue=0.7ms idle=1598.3ms
INSERT INTO "users" ("email","password_hash","username","inserted_at","updated_at") VALUES ($1,$2,$3,$4,$5) RETURNING "id" ["bobby.clarke@flyers.com", "$2b$12$yeeAXHE9gVOtvEFiJs85auzwBfjPCo/EJQ/8EDqZnbUTKR3HPhm/y", "bobby", ~N[2020-06-14 03:25:33], ~N[2020-06-14 03:25:33]]
{:ok,
 %Pucks.People.User{
   __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
   email: "bobby.clarke@flyers.com",
   id: 1,
   inserted_at: ~N[2020-06-14 03:25:33],
   password: "number16",
   password_confirmation: "number16",
   password_hash: "$2b$12$yeeAXHE9gVOtvEFiJs85auzwBfjPCo/EJQ/8EDqZnbUTKR3HPhm/y",
   updated_at: ~N[2020-06-14 03:25:33],
   username: "bobby"
 }}
```
Notice that only the *:username*, *:email*, and *:password_hash* field were inserted into the database.  We can double check this by listing all Users.

```zsh
iex(4)> Pucks.People.list_users

[debug] QUERY OK source="users" db=1.6ms queue=1.1ms idle=1450.6ms
SELECT u0."id", u0."email", u0."password_hash", u0."username", u0."inserted_at", u0."updated_at" FROM "users" AS u0 []
[
  %Pucks.People.User{
    __meta__: #Ecto.Schema.Metadata<:loaded, "users">,
    email: "bobby.clarke@flyers.com",
    id: 1,
    inserted_at: ~N[2020-06-14 03:25:33],
    password: nil,
    password_confirmation: nil,
    password_hash: "$2b$12$yeeAXHE9gVOtvEFiJs85auzwBfjPCo/EJQ/8EDqZnbUTKR3HPhm/y",
    updated_at: ~N[2020-06-14 03:25:33],
    username: "bobby"
  }
]
```
Let test out our validations with some flawed data:

```zsh
iex(5)> attrs = %{ email: "bobby.clarke@flyers.com", password: "number8", password_confirmation: "number24448"}
[debug] QUERY OK source="users" db=2.2ms idle=1901.8ms
SELECT TRUE FROM "users" AS u0 WHERE (u0."email" = $1) LIMIT 1 ["bobby.clarke@flyers.com"]
#Ecto.Changeset<
  action: nil,
  changes: %{
    email: "bobby.clarke@flyers.com",
    password: "number8",
    password_confirmation: "number24448",
    password_hash: "$2b$12$xGfFZRD4x4lQ8c.Y32dqMOErLwqurpYpPItUpMHOlvCtO/anbtr6u"
  },
  errors: [
    password_confirmation: {"does not match confirmation",
     [validation: :confirmation]},
    email: {"has already been taken",
     [validation: :unsafe_unique, fields: [:email]]},
    username: {"can't be blank", [validation: :required]}
  ],
  data: #Pucks.People.User<>,
  valid?: false
>
iex(6)>
```
As expected, it picked up the errors and reported all three, reporting the changeset was not valid (*valid?: flase*)
Let's clean up the information and add it to the database (results not shown for brevity).

```zsh
iex(6)> attrs = %{ email: "dave.schultz@flyers.com", username: "the hammer", password: "number8", password_confirmation: "number8"}
....

iex(7)> user = User.changeset(%User{}, attrs)

...

iex(8)> Pucks.Repo.insert user
```
For users, I am using the roster of the **1974 Stanley Cup Champion Philadephia Flyers** -- the famous *Broad Street Bullies*. A user's password is their jersey number --- so it makes it easy to remember.  I was six or seven when they won the cup.  I was not a fan of the team, but looking back, they were a very good team. Every one of them was Canadian as well. Just sayin'.

I added a some more so we have some records in our database.  I will leave that as an exercise for the reader.

### Working on the **UserController**

While we have made progress, we are not actually making and handling requests. Lets give our router, controller, and view some love. We will then use [Postman](https://www.postman.com) make requests.

In the UserController, we want to focus on the create method.

We will take the parameters passed in, call the ```People.create_user/1``` method which takes a struct (which is assigned to the changeset). It is *cast* and *validated* and, if it is a valid changeset it is save to the database (and returns ```{:ok, %User}``` tuple). We give the request a **201** status code (*:created*) and we return the user information in *JSON:API* format via the view (we will get to that soon).

If the changeset is not valid, we will get the ```{:error, _err}``` tuple (the second argument is and Ecto.changeset with the error messages).  In that case -- we issue a **401** status code (*:unprocessable_entity*), and send the errors, also in JSON:API compliant format.

Here is how the create method looks in the controller:

```elixir
# ./lib/pucks_web/controllers/user_controller.ex
#   ....
  def create(conn, %{"data" => data = %{"type" => "users", "attributes" => user_params}}) do
    attrs = JaSerializer.Params.to_attributes(data)

    case People.create_user(attrs) do
      {:ok, %User{} = user} ->
        conn
        |> put_status(:created)
        |> render("show.json-api", data: user)
      {:error, %Ecto.Changeset{} = changeset} ->
        conn
        |> put_status(:unprocessable_entity)
        |> put_view(PucksWeb.ErrorView)
        |> render("400.json-api", data: changeset)

  end
#   ....  
```
Now for the view files, which is where *ja_serializer* makes thing mostly trivial.

In the UserView, we use ```JaSerializer.PhoenixView```, set the location (which is the url setup) and the attributes we want serialized.  That's it.  Sure, it is a bit more complex when dealing with relationships and whatnot, but for this example it is simple.
```elixir
# ./lib/pucks_web/views/user_view.ex

defmodule PucksWeb.UserView do
  use PucksWeb, :view
  use JaSerializer.PhoenixView

  location "/users/:id"

  attributes [:username, :email]

end
```
The last thing we want prior to turning to Postman is to setup our *401.json* render method in our ErrorView module.
Thanks again to JaSerializer, and specifically the *EctoErrorSerializer.format* method, this render function is also super simple:

```elixir
# ./lib/pucks_web/views/error_view.ex

defmodule PucksWeb.ErrorView do
  use PucksWeb, :view
  use JaSerializer.PhoenixView

  def render("400.json-api", %{data: changeset}) do
    JaSerializer.EctoErrorSerializer.format(changeset)
  end
#   .....

end
```
Let's try it out in *Postman*. First off, you have set the request headers ```Accept``` and ```Content-Type``` for json-api (```application/vnd.api+json```).

Send a *GET* request to ```localhost:4000\api``` and see the list of users returned in JSON:API format.

Now try to create a new user with a *POST* - again, the ```Accept``` and ```Content-Type``` headers must be set, and the format of the body must be valid JSON:API. Here is an example of the body of the request to add [*Moose Dupont*](https://www.hockey-reference.com/players/d/duponan01.html).
```json
{    
	"data": {
		"type": "users",
        "attributes": {
        	"email": "andre.dupont@flyers.com",
            "username": "moose",
            "password": "number6",
            "password_confirmation": "number6"
        }
	}
 }
```
As you can see, the server responds with a **201** status code and a JSON:API formatted record for the new user.

Add a few more users.  Then try to delete one.  It seems to be working.

Let's see if the :error part of the case statement is working.  Try to create another Moose Dupont; this time with some errors (this also assumes you did not delete this user.)
```json
{    
	"data": {
		"type": "users",
        "attributes": {
        	"email": "andre.dupont@flyers.com",
            "password": "number6",
            "password_confirmation": "number44444"
        }
	}
 }
```
And our validations caught the mistakes, and ja_serializer put them in easily ingestable JSON:API.
```json
{
    "errors": [
        {
            "detail": "Password confirmation does not match confirmation",
            "source": {
                "pointer": "/data/attributes/password-confirmation"
            },
            "title": "does not match confirmation"
        },
        {
            "detail": "Email has already been taken",
            "source": {
                "pointer": "/data/attributes/email"
            },
            "title": "has already been taken"
        },
        {
            "detail": "Username can't be blank",
            "source": {
                "pointer": "/data/attributes/username"
            },
            "title": "can't be blank"
        }
    ],
    "jsonapi": {
        "version": "1.0"
    }
}
```
So we have completed adding a user to the sever side of the application.  Commit to the repo and move on.
```
git add .
git commit -am 'adding user with password hashing functionality implemented'
```
In [Part 2] - we turn our attention to our frontend, setup a basic Ember Octane app, and implement registering a new user.






