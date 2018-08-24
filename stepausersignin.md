---
layout: page
title: Add Member Signin
---

### Add a Signup Context

Now that we have a schedule let's add the ability for members to join the site - this will allow us to let people curate their own agenda at the conference.

We're going to have several user resources as part of this application - at the very least we'll have signup, login, and profile contexts. We'll start with signup first.

Paste the following in our commandline:

```
mix phx.gen.html Signup User users \
username:string:unique password:string --web Signup
```

This should seem rather familiar by now - it will generate all the files for a Signup.User context.  The `web` flag makes Phoenix generate a namespace for the web parts.  This will help us separate our authentication resources from our signup resources (more on that later).

The migration that was generated will ensure a unique username:

```
defmodule Fawkes.Repo.Migrations.CreateUsers do
  use Ecto.Migration

  def change do
    create table(:users) do
      add :username, :string
      add :password, :string

      timestamps()
    end

    create unique_index(:users, [:username])
  end
end
```

 We don't need to make any changes here so in the terminal run `mix ecto.mograte`.

The generated `Fawkes.Signup` context will contain a lot of things we don't need, however. For signup we will only need the `new` and `create` actions and related functionality - so let's delete everything except change_user/1 and create_user/1. Since we no longer run queries in this context we can also delete the `import Ecto.Query` line.  The end should look something like this:

```
defmodule Fawkes.Signup do
  @moduledoc """
  The Signup context.
  """

  alias Fawkes.Repo
  alias Fawkes.Signup.User

  @doc """
  Creates a user.

  ## Examples

      iex> create_user(%{field: value})
      {:ok, %User{}}

      iex> create_user(%{field: bad_value})
      {:error, %Ecto.Changeset{}}

  """
  def create_user(attrs \\ %{}) do
    %User{}
    |> User.changeset(attrs)
    |> Repo.insert()
  end

  @doc """
  Returns an `%Ecto.Changeset{}` for tracking user changes.

  ## Examples

      iex> change_user(user)
      %Ecto.Changeset{source: %User{}}

  """
  def change_user(%User{} = user) do
    User.changeset(user, %{})
  end
end
```

We'll do something similar to the controller - remove everything from `FawkesWeb.Signup.UserController` that isn't `new/2` or `create/2`. We'll also change the redirect on success in `create/2` to point to the schedule index page. It'll look like this:

```
defmodule FawkesWeb.Signup.UserController do
  use FawkesWeb, :controller

  alias Fawkes.Signup
  alias Fawkes.Signup.User

  def new(conn, _params) do
    changeset = Signup.change_user(%User{})
    render(conn, "new.html", changeset: changeset)
  end

  def create(conn, %{"user" => user_params}) do
    case Signup.create_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "User created successfully.")
        |> redirect(to: schedule_path(conn, :index))
      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end
end
```

Finally - we'll remove the templates we don't need - delete `templates/signup/user/edit.html.eex`, `templates/signup/user/index.html.eex`, and `templates/signup/user/show.html.eex`. There is also a `Back` link in our `templates/signup/user/new.html.eex` template that points to a route we're not going to have yet.  Let's just delete the line.  The result looks like this:

```
<h2>New User</h2>

<%= render "form.html", Map.put(assigns, :action, signup_user_path(@conn, :create)) %>
```

Now that we've removed what we don't need we can add what we do.

First - passwords should use an html input element with the `type=password` attribute.  Open `templates/signup/user/form.html.eex` and change our password `text_input` to `password_input`.

```
<%= password_input f, :password, class: "form-control" %>
```

Now we'll add the resources to our router - let's have a the url for a new user be "localhost:4000/signup/new". The generator suggests using a scope - we'll copy and paste the suggested code:

```
    scope "/signup", FawkesWeb.Signup, as: :signup do
      pipe_through :browser
      ...
      resources "/users", UserController
    end
```

This makes our url for `UserController` "localhost:4000/signup/users/new"... close - but let's remove the `users` segment.

```
      resources "/", UserController, only: [:new, :create]
```

Path helpers look at the names of modules (as opposed to the url strings) to name our paths - so this change will not affect anything the generator created.

### Hash and Validate Passwords

We can now signup new users! We can improve things a bit with some validators... and we really should hash our passwords before we store them. All that happens will happen in `Fawkes.Signup.User.changeset/2`.

We'll start by adding an encryption library.  There are a ton of great libraries available through hex (see [Riverrun's choosing the password algorithm](https://github.com/riverrun/comeonin/wiki/Choosing-the-password-hashing-algorithm) for examples).  We'll use bcrypt_elixir by adding it to our `mix.exs`:

```
  defp deps do
    [
      {:phoenix, "~> 1.3.2"},
      {:phoenix_pubsub, "~> 1.0"},
      {:phoenix_ecto, "~> 3.2"},
      {:postgrex, ">= 0.0.0"},
      {:phoenix_html, "~> 2.10"},
      {:phoenix_live_reload, "~> 1.0", only: :dev},
      {:gettext, "~> 0.11"},
      {:cowboy, "~> 1.0"},

      # For datetime formating
      {:timex, "~> 3.3"},

      # For authentication
      {:bcrypt_elixir, "~> 1.0"},
    ]
  end
```

... and runing `mix deps.get` on our commandline.

Now we'll edit our Fawkes.Signup.User.changeset/2 - add a step in our pipeline which calls a private method to replace our plain text password with a hashed version.

```
  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:username, :password])
    |> validate_required([:username, :password])
    |> unique_constraint(:username)
    |> put_pass_hash()
  end
```

This method takes and returns a changeset. We can transform a changeset using `Ecto.Changeset.change/2` (available as just `change/2` because we import `Ecto.Changeset`). We'll use a pattern match in the method signature to so that when a changeset is valid we hash the password with `Comeonin.Bcrypt.hashpwsalt/1`.  If the pattern match fails we'll change the password to an empty string:

```
  defp put_pass_hash(%{valid?: true, changes: params} = changeset) do
    password = Comeonin.Bcrypt.hashpwsalt(params[:password])
    change(changeset, password: password)
  end

  defp put_pass_hash(changeset) do
    change(changeset, password: "")
  end
```

While we're here let's add a few other useful validators to our changeset - we can do things like blacklist passwords with `validate_exclusion` or ensure the length of the password with `validate_length`. The resulting schema will look something like this:

```
defmodule Fawkes.Signup.User do
  use Ecto.Schema
  import Ecto.Changeset
  alias Comeonin.Bcrypt

  @bad_passwords ~w(
    12345678
    password1
    qwertyuiop
  )

  schema "users" do
    field :password, :string
    field :username, :string

    timestamps()
  end

  @doc false
  def changeset(user, attrs) do
    user
    |> cast(attrs, [:username, :password])
    |> validate_required([:username, :password])
    |> unique_constraint(:username)
    |> validate_exclusion(
         :password,
         @bad_passwords,
         message: "That password is too common.")
    |> validate_length(:password, min: 8)
    |> put_pass_hash()
  end

  defp put_pass_hash(%{valid?: true, changes: params} = changeset) do
    password = Comeonin.Bcrypt.hashpwsalt(params[:password])
    change(changeset, password: password)
  end

  defp put_pass_hash(changeset) do
    change(changeset, password: "")
  end
end
```

You can read more on validations on the [Ecto.Changeset Hexdocs](https://hexdocs.pm/ecto/Ecto.Changeset.html).

### Add an Auth Context

Now that we can sign up we need to be able to log in. The process will be almost identical.  Start with the generator:

```
mix phx.gen.html Auth User users \
username:string password:string --web Auth
```

This time we can delete the migration (since our new context will just use the same users table we generated before.)

We don't create anything when a user logs in - so delete everything except `change_user/1` from the `Fawkes.Auth` context.

```
defmodule Fawkes.Auth do
  @moduledoc """
  The Auth context.
  """

  alias Fawkes.Auth.User

  @doc """
  Returns an `%Ecto.Changeset{}` for tracking user changes.

  ## Examples

      iex> change_user(user)
      %Ecto.Changeset{source: %User{}}

  """
  def change_user(%User{} = user) do
    User.changeset(user, %{})
  end
end
```

Remove everything but `new/2` and `create/2` from `FawkesWeb.Auth.UserController`.  (We'll eventually need a `delete/2`) but the implementation is different so we'll delete the generated default. Change the redirect on successs in `create/2` to the schedule index.  The result will look like:

```
defmodule FawkesWeb.Auth.UserController do
  use FawkesWeb, :controller

  alias Fawkes.Auth
  alias Fawkes.Auth.User

  def new(conn, _params) do
    changeset = Auth.change_user(%User{})
    render(conn, "new.html", changeset: changeset)
  end

  def create(conn, %{"user" => user_params}) do
    case Auth.create_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "User created successfully.")
        |> redirect(to: schedule_path(conn, :index))
      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end
end

```

Delete `templates/auth/user/edit.html.eex`, `templates/auth/user/index.html.eex`, and `templates/auth/user/show.html.eex`. Remove the `Back` link in our `templates/auth/user/new.html.eex`

```
<h2>New User</h2>

<%= render "form.html", Map.put(assigns, :action, auth_user_path(@conn, :create)) %>

```

Change the input in `templates/auth/user/form.html.eex`

```
<%= password_input f, :password, class: "form-control" %>
```

Next we'll return to the Fawkes.Auth context and add an `authenticate_user/2` method.  This method contain a pipeline to that takes the params and returns a boolean indicating whether the user/password combo is authentic.

```
def authenticate_user(%{"username" => user, "password" => pass}) do
  user
  |> fetch_user_by username()
  |> check_password(pass)
end
```

`fetch_user_by username()` is a simple ecto query:

```
import Ecto.Query, warn: false
alias Fawkes.Repo

# . . .

def fetch_user_by_username(username) do
  User
  |> where([user], user.username == ^username)
  |> Repo.one
end
```

For `check_password/2` We'll pattern match to ensure that if we don't have a user we just return false:

```
  defp check_password(%User{} = user, password) do
    # Check the password
  end

  defp check_password(_, _), do: {:error, :incorrect}
```

... and then leverage `Comeonin.Bcrypt.checkpw/2` to check the given password against the password on the found user.  If it succeeds we'll return an :ok tagged tuple... otherwise we'll return an :error tagged tuple after running `Comeonin.Bcrypt.dummy_checkpw/0`.  (`dummy_checkpw/0` defends against [timing attacks](https://en.wikipedia.org/wiki/Timing_attack).)

```
  defp check_password(%User{} = user, password) do
    password
    |> Bcrypt.checkpw(user.password)
    |> case do
      true ->
        {:ok, user}
      false ->
        Bcrypt.dummy_checkpw()
        {:error, :incorrect}
    end
  end
```

Our final result will look something like this:

```
defmodule Fawkes.Auth do
  @moduledoc """
  The Auth context.
  """

  import Ecto.Query, warn: false
  alias Comeonin.Bcrypt
  alias Fawkes.Auth.User
  alias Fawkes.Repo

  @doc """
  Returns an `%Ecto.Changeset{}` for tracking user changes.

  ## Examples

      iex> change_user(user)
      %Ecto.Changeset{source: %User{}}

  """
  def change_user(%User{} = user) do
    User.changeset(user, %{})
  end

  def authenticate_user(%{"username" => user, "password" => pass}) do
    user
    |> fetch_user_by username()
    |> check_password(pass)
  end

  defp fetch_user_by_username(username) do
    User
    |> where([user], user.username == ^username)
    |> Repo.one
  end

  defp check_password(%User{} = user, password) do
    password
    |> Bcrypt.checkpw(user.password)
    |> case do
      true ->
        {:ok, user}
      false ->
        Bcrypt.dummy_checkpw()
        {:error, :incorrect}
    end
  end

  defp check_password(_, _), do: false
end
```

Now we can change our `FawkesWeb.Auth.UserController` to leverage `authenticate_user/1` (instead of the non-existent `Auth.create_user/1`)

```
defmodule FawkesWeb.Auth.UserController do
  use FawkesWeb, :controller

  alias Fawkes.Auth
  alias Fawkes.Auth.User

  def new(conn, _params) do
    changeset = Auth.change_user(%User{})
    render(conn, "new.html", changeset: changeset)
  end

  def create(conn, %{"user" => user_params}) do
    case Auth.authenticate_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "User created successfully.")
        |> redirect(to: schedule_path(conn, :index))
      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end
end

```

### Add a Membership Context

Now that we're able to assess whether a user is authentic we need to make that information available to our controllers somehow. We need a user data type to leverage outside the Auth and Signup contexts.  This new user context will lack access to `password` and will give us a place to relate profiles and agendas without muddying the data types we use to signin and authenticate. To pull this off we'll generate a new `Membership` context.  First the `User`:

```
defmodule Fawkes.Membership.User do
  use Ecto.Schema

  schema "users" do
    field(:username, :string)

    timestamps()
  end
end
```

And then a context with a method to find those users based on their id:

```
defmodule Fawkes.Membership do
  @moduledoc """
  Context responsible for managing profile information
  """

  import Ecto.Query
  alias Fawkes.Membership.User
  alias Fawkes.Repo

  def get_user(id) do
    User
    |> where([user], user.id == ^id)
    |> Repo.one()
  end
end
```

### Sign and Verify (or reject) Authentication Tokens

Now that we've defined `Fawkes.Membership.User` to use throughout the app - we need to figure out how to keep that data handy in a session once the user has authenticated.  For that we'll use Guardian - library that signs and verifies [JSON Web Tokens](https://jwt.io/). That topic is _deep_ so we're gonna accept the magic here. For now - Guardian is how we'll log people in. We'll add it (and Comeonin as a dependency) to our `mix.exs` file.

```
defp deps do
  [
    {:phoenix, "~> 1.3.2"},
    {:phoenix_pubsub, "~> 1.0"},
    {:phoenix_ecto, "~> 3.2"},
    {:postgrex, ">= 0.0.0"},
    {:phoenix_html, "~> 2.10"},
    {:phoenix_live_reload, "~> 1.0", only: :dev},
    {:gettext, "~> 0.11"},
    {:cowboy, "~> 1.0"},

    # For datetime formating
    {:timex, "~> 3.3"},

    # For authentication
    {:bcrypt_elixir, "~> 1.0"},
    {:comeonin, "~> 4.0"},
    {:guardian, "~> 1.0"},
  ]
end
```

...then run `mix deps.get` in our terminal.

Guardian requires a bit of setup.

First we need to create something to parse token information.  According to the documentation this module needs to `use` Guardian and implement two methods.  `subject_for_token/2` needs to generate a value in session which can later be used to look up a user.  We'll use the user's id so we can leverage that `Fawkes.Membership.get_user/1` method we created.  Then we need a `resource_from_claims/1` which takes the id (called a "sub" ... short for "subject") and returns a Fawkes.Membership.User.  We can then access that user in our application.

This module will be responsible for translating iodata from the web - so let's create `FawkesWeb.Guardian.Tokenizer` in the `lib/fawkes_web/guardian` folder - adding `use Guardian, otp_app: <app_name>` the way the documentation asks:

```
defmodule FawkesWeb.Guardian.Tokenizer do
  use Guardian, otp_app: :fawkes
end
```

Then add to it our `subject_for_token/2`.  Later on we'll hand it a Signup or Auth User and it needs to return an :ok tagged tuple containing the id of the given user as a string.:

```
  def subject_for_token(%{id: id}, _) do
    {:ok, to_string(id)}
  end
```

Then we add `resource_from_claims/1` - this will need to return with an :ok tagged tuple with the Membership User or an :error tagged tuple.  Guardian will hand it a JWT "claim" - which will (by convention) store our token subject (the id) in a key called `"sub"`.

```
def resource_from_claims(claims) do
  case Membership.get_user(claims["sub"]) do
    nil -> {:error, :resource_not_found}
    user -> {:ok, user}
  end
end
```

The resulting module should look like this:

```
defmodule FawkesWeb.Guardian.Tokenizer do
  use Guardian, otp_app: :fawkes
  alias Fawkes.Membership

  def subject_for_token(%{id: id}, _) do
    {:ok, to_string(id)}
  end

  def resource_from_claims(claims) do
    case Membership.get_user(claims["sub"]) do
      nil -> {:error, :resource_not_found}
      user -> {:ok, user}
    end
  end
end
```

Next we have some configuration to do - handing guardian a secret key to use for encryption and telling it where our Token logic lives. Run `mix phx.gen.secret` to get a random string to use for this - then in both `config/dev.exs` and `config/test.exs` add the following:

```
config :fawkes, FawkesWeb.Guardian.Tokenizer,
                issuer: "fawkes",
                secret_key: "our random string"
```

Then in `config/prod.exs` we'll want to leverage environment vars for our secret key for security:

```
config :fawkes,
       FawkesWeb.Guardian.Tokenizer,
       issuer: "fawkes",
       secret_key: Map.fetch!(System.get_env(),
                              ”GUARDIAN_KEY")
```

(If you've got phx.server running you'll need to restart it to see this change.)

Now we're ready to sign our JWT.  In `FawkesWeb.Auth.UserController` we'll do that by calling a method from a meta-programmed module Guardian creates called `FawkesWeb.Guardian.Tokenizer.Plug.sign_in/2`.  The result (after some alias goodness) looks like this:

```
defmodule FawkesWeb.Auth.UserController do
  use FawkesWeb, :controller

  alias Fawkes.Auth
  alias Fawkes.Auth.User
  alias FawkesWeb.Guardian.Tokenizer.Plug

  def new(conn, _params) do
    changeset = Auth.change_user(%User{})
    render(conn, "new.html", changeset: changeset)
  end

  def create(conn, %{"user" => user_params}) do
    case Auth.create_user(user_params) do
      {:ok, user} ->
        conn
        |> put_flash(:info, "User created successfully.")
        |> GuardianPlug.sign_in(user)
        |> redirect(to: auth_user_path(conn, :show, user))
      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "new.html", changeset: changeset)
    end
  end
end
```

While we're here let's add a `delete/2` action to allow us to logout. This will call `FawkesWeb.Guardian.Tokenizer.Plug.sign_out/2`

```
def delete(conn, _) do
  conn
  |> GuardianPlug.sign_out()
  |> redirect(to: page_path(conn, :index))
end
```

We'll come back to delete later - for now let's add login new and create to our routes:

```
  scope "/login", FawkesWeb.Auth, as: :auth do
    pipe_through :browser

    resources "/", UserController, only: [:new, :create]
  end

```

### Adding a Plug Pipelines for Authentication

Ok - now we can sign and unsign authentication tokens... but to make the resource available we need to add a plug.

Plug is the core of the Phoenix Framework. It defines a type - called a Connection Struct (or conn) which is passed through a "plug pipeline". The connection struct holds the state of the request - and is transformed at each step of the pipeline. For most pages on the site we want Guardian to add verify the JWT we created during signup and then add it to the conn for later use.

Ok... a personal note - Phoenix isn't super magical... but Guardian kinda is. Getting in to why this thing works the way it does is probably not "basic" - so I'm gonna get hand wavy here.  Follow along and it will probably make more sense in a couple minutes.  Ask questions if you have them - and plan to come back and read the documentation for Guardian at some point in the future to get a better understanding.

To do that we first need to tell Guardian how to handle an invalid JWT. Following the documentation - this module we generate needs to implement an `auth_error/3` method. We'll jsut use the boilerplate provided by the documentation and add the following to `lib/fawkes_web/guardian/error_handler.ex`:

```
defmodule FawkesWeb.Guardian.ErrorHandler do
  @moduledoc """
  Logic for handling errors during the auth process
  """

  import Plug.Conn

  def auth_error(conn, {type, _reason}, _opts) do
    conn
    |> put_resp_content_type("text/plain")
    |> send_resp(401, to_string(type))
  end
end
```

Now we can generate a module to act as plug pipeline for Guardian to verify tokens. Following documentation we'll add a `lib/fawkes_web/guardian/plug.ex`

```
defmodule FawkesWeb.Guardian.Plug do
  @moduledoc """
  Pipeline which ensures a user is authenticated
  """

  use Guardian.Plug.Pipeline,
      otp_app: :fawkes,
      error_handler: FawkesWeb.Guardian.ErrorHandler,
      module: FawkesWeb.Guardian.Tokenizer

  # If there is a session token, validate it
  plug(Guardian.Plug.VerifySession, claims: %{"typ" => "access"})

  # If there is an authorization header, validate it
  plug(Guardian.Plug.VerifyHeader, claims: %{"typ" => "access"})

  # Load the user if either of the verifications worked
  plug(Guardian.Plug.LoadResource, allow_blank: true)
end
```

Adding this plug to a pipeline (back to that in a sec) will verify the request and stash the information in our connection struct.  We'll have a method available, `FawkesWeb.Guardian.Tokenizer.Plug.current_resource/1`, which takes our conn and returns the Membership User. We can generate a plug of our own (in `lib/fawkes_web/guardian/current_user_plug.ex`) to place this in an easier-to-access location using Plug.Conn.assign/3:

```
defmodule FawkesWeb.Guardian.CurrentUserPlug do
  @moduledoc """
  Plug that populates the current_user assigns
  """

  alias FawkesWeb.Guardian.Tokenizer.Plug,
        as: GuardianPlug
  alias Plug.Conn

  def init(opts), do: opts

  def call(conn, _opts) do
    Conn.assign(conn, :current_user,
                GuardianPlug.current_resource(conn))
  end
end
```

Once we use this we can get our current_user by using `conn.assigns.current_user`.

### Using Our New Pipelines in the Router

That was a lot - let's set up our router and then show it in action. In the router we'll create a new pipeline called guardian which uses the two plugs we just created to verify JWTs:

```
  pipeline :guardian do
    plug FawkesWeb.Guardian.Plug
    plug FawkesWeb.Guardian.CurrentUserPlug
  end
```

Then we'll create another pipeline to force authentication - this will leverage a default plug from Guardian which will use the error handler module we created if no verified JWT is found.

```
  pipeline :ensure_auth do
    plug Guardian.Plug.EnsureAuthenticated
  end
```

The order of these is important - we'll need our :ensure_auth pipeline to only run _after_ the :guardian pipeline (otherwise the connection struct will not have a verified token yet).  We'll only want to let users logout if they are authenticated - so let's try that out with a logout route:

```
  scope "/", FawkesWeb do
    pipe_through [:browser, :guardian, :ensure_auth]
    post("/logout", Auth.UserController, :delete)
  end
```

And add just the guardian pipeline to the rest of our routes (so we have current_user available)

```
  scope "/", FawkesWeb do
    pipe_through [:browser, :guardian]

    # ...
  end

  scope "/signup", FawkesWeb.Signup, as: :signup do
    pipe_through [:browser, :guardian]

    # ...
  end

  scope "/login", FawkesWeb.Auth, as: :auth do
    pipe_through [:browser, :guardian]

    # ...
  end
```

The result should look like this:

```
defmodule FawkesWeb.Router do
  use FawkesWeb, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  pipeline :guardian do
    plug FawkesWeb.Guardian.Plug
    plug FawkesWeb.Guardian.CurrentUserPlug
  end

  pipeline :ensure_auth do
    plug Guardian.Plug.EnsureAuthenticated
  end

  scope "/", FawkesWeb do
    pipe_through [:browser, :guardian, :ensure_auth]
    post("/logout", PageController, :logout)
  end

  scope "/", FawkesWeb do
    pipe_through [:browser, :guardian]

    get "/", PageController, :index

    resources("/schedule", ScheduleController, only: [:index, :show])
    resources("/audience", AudienceController, only: [:show])
    resources("/category", CategoryController, only: [:show])
    resources("/location", LocationController, only: [:show])
    resources("/speaker", SpeakerController, only: [:index, :show])
    resources("/talk", TalkController, only: [:show])
  end

  scope "/signup", FawkesWeb.Signup, as: :signup do
    pipe_through [:browser, :guardian]

    resources "/", UserController, only: [:new, :create]
  end

  scope "/login", FawkesWeb.Auth, as: :auth do
    pipe_through [:browser, :guardian]

    resources "/", UserController, only: [:new, :create]
  end

  # Other scopes may use custom stacks.
  # scope "/api", FawkesWeb do
  #   pipe_through :api
  # end
end
```

### Adding Authentication Links to the Layout

Finally we need to add links so that users can access signup, login and logout globally.  In our `lib/fawkes_web/templates/layout/app.html.eex` render a partial.

```
  <header class="header">
    <nav role="navigation">
      <ul class="nav nav-pills pull-right">
        <%= render(__MODULE__, "session_list_items.html", conn: @conn) %>
      </ul>
    </nav>
    <span class="logo"></span>
  </header>
```

and then add our signup and/o auth links in `lib/fawkes_web/templates/layout/session_list_items.html.eex`:

```
<li>
  <%= if is_nil @conn.assigns[:current_user] do %>
    <li><%= link gettext("Get Started"), to: signup_user_path(@conn, :new) %></li>
    <li><%= link gettext("Log In"), to: auth_user_path(@conn, :new) %></li>
  <% else %>
    <%= link gettext("Logout"), to: auth_user_path(@conn, :delete), method: :post%>
  <% end %>
</li>
```
