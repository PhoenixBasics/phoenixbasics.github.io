---
layout: page
title: Add User Profiles
---

### Set up image uploads with S3

Before we get started - we'll want to be able to upload images for users as part of their profiles.  Since we're deploying to Heroku (and Heroku doesn't have local file storage on our tier) we'll need to work out some other solution.  In this case - we'll set up our application to leverage S3.

First we'll add arc and all it's dependencies to our `mix.exs` file:

```
    # For S3 integration and image uploads
    {:arc, "~> 0.10"},       # File upload
    {:arc_ecto, "~> 0.10"},  # Ecto Type
    {:ex_aws, "~> 2.0"},     # Aws
    {:ex_aws_s3, "~> 2.0"},  # S3
    {:hackney, "~> 1.9"},    # API HTTP client
    {:poison, "~> 3.0"},     # Fast JSON Parser
    {:sweet_xml, "~> 0.6"}   # XML Parsing
```

AWS is going to require us to add our keys to config.  We don't want to add thet to source control so we'll set up some secret configs.  At the bottom of `config/dev.exs` add the following line:

```
import_config "dev.secret.exs"
```

Then we'll create that file in the same folder.  To it we'll add the following:

```
use Mix.Config

config :arc, bucket: "bucket_name"

config :ex_aws,
  access_key_id: ["key_id", :instance_role],
  secret_access_key: ["secret_key", :instance_role]
```

... where "bucket_name" is the name of your s3 bucket, and "key_id" and "secret_key" are your credentials from aws.

Repeat that process for test.  Similar to before; for production we'll want to handle this through env vars.  Add teh following to `config/prod.exs`.

```
config :arc,
       bucket: Map.fetch!(System.get_env(),  “S3_BUCKET")

config :ex_aws,
  access_key_id: [Map.fetch!(System.get_env(), "AWS_ACCESS_KEY"), :instance_role],
  secret_access_key: [Map.fetch!(System.get_env(), "AWS_SECRET_KEY"), :instance_role]
```

Now that we have S3 configured we need to add an Arc definition module to our project.  You'll want to look at docs for this file as it's where you'd define things like thumbnail resizing and renaming. For now we'll add a simple one.  In `lib/fawkes/uploader/image.ex` add the following.

```
defmodule Fawkes.Uploader.Image do
  @moduledoc false

  use Arc.Definition
  use Arc.Ecto.Definition

  @versions [:original]
end
```

This is more or less based on the generator mentioned in docs for this file - however Arc hasn't been updated to the file structure used in newer versions of Phoenix (someone get on that for me...).

### Add S3 images to speaker profiles

Now we can try this out.  Currently we have speakers which leverage the profiles table to get their data.  We'll reuse this table for user info - so let's try out image uploads on the Speaker seeds.

First we'll need to open up `Fawkes.Schedule.Speaker`.  To it we'll add a call to `use Arc.Ecto.Schema` just after our call to use Ecto.Schema.  We'll add or change the image field to have a type that was meta programmed by our uploader:

```
field(:image, Fawkes.Uploader.Image.Type)
```

Then we'll need to add the `image` param to our changeset along with a call to `cast_attachments/3`:

```
  @doc false
  def changeset(speaker, attrs) do
    speaker
    |> cast(attrs, [:slug, :image, :first, :last, :company, :github, :twitter, :description])
    |> cast_attachments(attrs, [:image])
    |> validate_required([:slug, :image, :first, :last, :description])
    |> unique_constraint(:slug)
  end
```

Our schema is now using S3 to store images!  We can try this out with the seeds. Go in to `lib/fawkes/schedule/seed/speaker.ex`.  You'll find the images all commented out. Let's uncomment those and then run `mix ecto.reset`.

Our speaker images are now available on S3!  Now we just need to add them to the templates. Let's open the speaker/show.html.eex page and add the following:

```
<%=
  {@speaker.image, @speaker}
  |> Fawkes.Uploader.Image.url()
  |> img_tag()
%>
```

`Fawkes.Uploader.Image.url()` takes that tuple and generates the correct url to use to grab the image.

### Add the profile schema

Our membership profile schema will look very similar to the speaker with the addition of a `belongs_to` statement:

```
defmodule Fawkes.Membership.Profile do
  @moduledoc false

  use Ecto.Schema
  use Arc.Ecto.Schema
  import Ecto.Changeset
  alias Fawkes.Repo.Symbol, as: SymbolType

  schema "profiles" do
    field(:company, :string)
    field(:description, :string)
    field(:first, :string)
    field(:github, :string)
    field(:image, Fawkes.ImageUploader.Type)
    field(:last, :string)
    field(:slug, SymbolType)
    field(:twitter, :string)
    field(:title, :string)

    belongs_to(:user, Fawkes.Profile.User)

    timestamps()
  end
end

```

We'll want to add `has_one(:profile, Fawkes.Membership.Profile)` to our Membership.User schema. We'll need a migration to complete that task - in terminal run `mix ecto.gen.migration add_user_id_to_profiles` and then add our alter table statement:

```
defmodule Fawkes.Repo.Migrations.AddUserIdToProfiles do
  use Ecto.Migration

  def change do
    alter table("profiles") do
      add :user_id, references("users")
    end
  end
end

```

And run `mix ecto.migrate`.

We'll also want to preload our profile any time we use `Fawkes.Membership.get_user/1` to ensure our `current_user` has the info:

```
defmodule Fawkes.Membership do
  import Ecto.Query
  alias Fawkes.Membership.User
  alias Fawkes.Repo

  def get_user(id) do
    User
    |> preload([:profile])
    |> where([user], user.id == ^id)
    |> Repo.one()
  end
end

```

Next let's add a method that will find or create a changeset for a user profile.  In `Fawkes.Membership` we'll two versions of profile_changeset - each using a different implementation of changeset on Profile.

```
  defp profile_changeset(%User{profile: profile}) when not is_nil(profile) do
    Profile.changeset(profile, %{})
  end

  defp profile_changeset(user)
    %Profile{}
    |> Profile.init_changeset(%{user_id: user.id, slug: "user_#{user.id}"})
    |> Repo.insert!()
    |> Profile.changeset(%{})
  end
```

Our first changeset will only accept a user_id and a slug.  Our second will look more like the changeset on `Schedule.Speaker` -  casting everything except for user_id.

Now when the user passed to profile_changeset/1 has a nil profile we generate a new one with the slug prepopulated.  (We'll let users personalize this if they want later).  While we're here let's also generate a `profile_update/2` method.

```
  def update_profile(%User{profile: profile}, params) when not is_nil(profile) do
    profile
    |> Profile.changeset(params)
    |> Repo.update()
  end
```

The result should look like this:

```
defmodule Fawkes.Membership do
  @moduledoc """
  Context responsible for managing profile information
  """

  import Ecto.Query
  alias Fawkes.Membership.User
  alias Fawkes.Membership.Profile
  alias Fawkes.Repo

  def get_user(id) do
    User
    |> where([user], user.id == ^id)
    |> preload([:profile])
    |> Repo.one()
  end

  def profile_changeset(%User{profile: profile}) when not is_nil(profile) do
    Profile.changeset(profile, %{})
  end

  def profile_changeset(user) do
    %Profile{}
    |> Profile.init_changeset(%{user_id: user.id, slug: "user_#{user.id}"})
    |> Repo.insert!()
    |> Profile.changeset(%{})
  end

  def update_profile(%User{profile: profile}, params) when not is_nil(profile) do
    profile
    |> Profile.changeset(params)
    |> Repo.update()
  end
end

```

We'll also need to add our changesets to `Fawkes.Membership.Profile`:

```
  @doc false
  def init_changeset(info, attrs) do
    info
    |> cast(attrs, [:user_id, :slug])
    |> validate_required([:user_id, :slug])
    |> unique_constraint(:user_id)
    |> unique_constraint(:slug)
  end

  @doc false
  def changeset(info, attrs) do
    info
    |> cast(attrs, [:first, :last, :slug, :company, :title, :github, :twitter,
                    :description, :image])
    |> cast_attachments(attrs, [:image])
    |> validate_required([:first, :last])
    |> validate_exclusion(:slug, [:edit, :new])
    |> unique_constraint(:slug)
  end
```

### Add profile edit page

First we'll need a profile controller. For now we'll just implement the Edit and Update actions.

```
defmodule FawkesWeb.ProfileController do
  use FawkesWeb, :controller

  alias Fawkes.Membership

  def edit(conn, _params) do
    render(conn, "edit.html", changeset: Membership.profile_changeset(conn.assigns.current_user))
  end

  def update(conn, %{"info" => params}) do
    case Membership.update_profile(conn.assigns.current_user, params) do
      {:ok, _} ->
        conn
        |> put_flash(:info, "User updated successfully.")
        |> redirect(to: schedule_path(conn, :index))

      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "edit.html", changeset: changeset)
    end
  end
end


```

We'll need a ProfileView:

```
defmodule FawkesWeb.ProfileView do
  use FawkesWeb, :view
end

```

And the form template itself (with `multipart: true` set) in an edit.html.eex:

```
<h2>Edit User</h2>

<%= form_for @changeset, profile_path(@conn, :update), [multipart: true], fn f -> %>
  <%= if @changeset.action do %>
    <div class="alert alert-danger">
      <p><%= gettext("Oops, something went wrong! Please check the errors below.") %></p>
    </div>
  <% end %>

  <div class="form-group">
    <%=
      label(f, :first, class: "control-label") do
        gettext("First Name")
      end
    %>
    <%= text_input f, :first, class: "form-control" %>
    <%= error_tag f, :first %>
  </div>

  <div class="form-group">
    <%=
      label(f, :last, class: "control-label") do
        gettext("Last Name")
      end
    %>
    <%= text_input f, :last, class: "form-control" %>
    <%= error_tag f, :last %>
  </div>

  <div class="form-group">
    <%=
      label(f, :image, class: "control-label") do
        gettext("Picture")
      end
    %>
    <%= file_input f, :image, class: "form-control" %>
    <%= error_tag f, :image %>
  </div>

  <div class="form-group">
    <%=
      label(f, :slug, class: "control-label") do
        gettext("Profile URL")
      end
    %>
    <%= text_input f, :slug, class: "form-control" %>
    <%= error_tag f, :slug %>
  </div>

  <div class="form-group">
    <%=
      label(f, :company, class: "control-label") do
        gettext("Company")
      end
    %>
    <%= text_input f, :company, class: "form-control" %>
    <%= error_tag f, :company %>
  </div>

  <div class="form-group">
    <%=
      label(f, :title, class: "control-label") do
        gettext("Title")
      end
    %>
    <%= text_input f, :title, class: "form-control" %>
    <%= error_tag f, :title %>
  </div>

  <div class="form-group">
    <%=
      label(f, :github, class: "control-label") do
        gettext("Github")
      end
    %>
    <%= text_input f, :github, class: "form-control" %>
    <%= error_tag f, :github %>
  </div>

  <div class="form-group">
    <%=
      label(f, :twitter, class: "control-label") do
        gettext("Twitter")
      end
    %>
    <%= text_input f, :twitter, class: "form-control" %>
    <%= error_tag f, :twitter %>
  </div>

  <div class="form-group">
    <%=
      label(f, :description, class: "control-label") do
        gettext("Bio")
      end
    %>
    <%= textarea f, :description, class: "form-control" %>
    <%= error_tag f, :description %>
  </div>

  <div class="form-group">
    <%= submit gettext("Submit"), class: "btn btn-primary" %>
  </div>
<% end %>

```

And add them to our router where we use our ensure_auth pipeline. We'll add this as a singleton so that our url to edit teh current_users profile is just "/profile/edit"

```
  scope "/", FawkesWeb, as: :membership do
    pipe_through [:browser, :guardian, :ensure_auth]

    resources("/profile", ProfileController, only: [:edit, :update], singleton: true)
  end
```

Next we'll add a user index so we can see all the user profiles at the conference.  We'll ass index and show actions to our profile controller:

```
  def index(conn, _) do
    render(conn, "index.html", profiles: Membership.fetch_user_profiles())
  end

  def show(conn, %{"id" => slug}) do
    render(conn, "show.html", user: Membership.fetch_user_profile(slug))
  end
```

Note that we plan to look up users by the slug on profile.  We'll need to add those fetch methods to our Membership context:

```
  def fetch_user_profiles do
    Profile
    |> order_by([profile], profile.last)
    |> Repo.all()
  end

  def fetch_user_profile(slug) do
    Profile
    |> where([profile], profile.slug == ^slug)
    |> Repo.one()
  end
```

... and we'll need to add an `index.html.eex`

```
<ul>
  <%= for profile <- @profiles do %>
  <li><%= link("#{profile.first} #{profile.last}", to: profile_path(@conn, :show, profile.slug)) %></li>
  <% end %>
</ul>

```

and a `show.html.eex`

```
<div>
  <%= {@user.image, @user} |> Fawkes.Uploader.Image.url() |> img_tag() %>
  <h4><%= "#{@user.first} #{@user.last}" %></h4>

  <%= unless is_nil(@user.github) && is_nil(@user.twitter) && is_nil(@user.company) && is_nil(@user.title) do %>
    <ul>
      <%= unless is_nil(@user.github) do %>
      <li>Github: <%= link(@user.github, to: "https://github.com/#{@user.github}/") %></li>
      <% end %>
      <%= unless is_nil(@user.twitter) do %>
      <li>Twitter: <%= link(@user.twitter, to: "https://twitter.com/#{@user.twitter}/") %></li>
      <% end %>
      <%= unless @user.title |> FawkesWeb.SharedView.employment(@user.company) |> is_nil do %>
      <li><%= FawkesWeb.SharedView.employment(@user.title, @user.company) %></li>
      <% end %>
    </ul>
  <% end %>

  <div><%= @user.description %></div>
</div>

```

(Which can easily be DRYed up with template partials)


We'll also need to add routes for these pages. We can add them to the main pipeline since we don't need to be logged in to see them.

```
  scope "/", FawkesWeb do
    pipe_through [:browser, :guardian]

    get "/", PageController, :index

    resources("/schedule", ScheduleController, only: [:index, :show])
    resources("/audience", AudienceController, only: [:show])
    resources("/category", CategoryController, only: [:show])
    resources("/location", LocationController, only: [:show])
    resources("/speaker", SpeakerController, only: [:index, :show])
    resources("/talk", TalkController, only: [:show])
    resources("/member", ProfileController, only: [:index, :show])
  end
```

Now if we visit your profile page (most likely http://localhost:4000/member/user_1) you'll see your profile with the image uploaded to S3.
