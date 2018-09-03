---
layout: page
title: Add User Profiles
---

### Set up image uploads with S3
(Use `git checkout 7.user_profile` to catch up with the class)

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
       bucket: Map.fetch!(System.get_env(),  "S3_BUCKET")

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
(Use `git checkout 7a.speaker_image` to catch up with the class)

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

Our speaker images are now available on S3!  Now we just need to add them to the templates. Let's open the `speaker/show.html.eex` page replace the HTML with this:

```
<div class="container speaker-details">
  <div class="card details">
    <div class="card-body">
      <img src="<%= Fawkes.Uploader.Image.url({@speaker.image, @speaker}) %>" class="rounded-circle speaker-details__pic">
      <div class="clearfix"></div>
      <div class="speaker__info">
        <h4><%= FawkesWeb.SharedView.full_name(@speaker) %></h4>
        <div class="speaker__handle">
          <i class="fab fa-github"></i> @<%= @speaker.github %>
        </div>
        <div class="speaker__handle">
          <i class="fab fa-twitter"></i> @<%= @speaker.twitter %>
        </div>
      </div>
    </div>
  </div>

  <hr />
  <h4 class="subheader">About</h4>
  <p><%= @speaker.description %></p>

</div>
```


Note on line 4, it calls `Fawkes.Uploader.Image.url()`, which takes the tuple and generates the correct image url.


### Add the profile schema
(Use `git checkout 7b.profile_schema` to catch up with the class)

Our membership profile schema will look very similar to the speaker with the addition of a `belongs_to` statement:

```
defmodule Fawkes.Membership.Profile do
  @moduledoc false

  use Ecto.Schema
  use Arc.Ecto.Schema
  import Ecto.Changeset

  schema "profiles" do
    field(:company, :string)
    field(:description, :string)
    field(:first, :string)
    field(:github, :string)
    field(:image, Fawkes.Uploader.Image.Type)
    field(:last, :string)
    field :slug, :string
    field(:twitter, :string)

    belongs_to(:user, Fawkes.Profile.User)

    timestamps()
  end
end
```

Open `lib/fawkes/membership/user.ex`. Add `has_one(:profile, Fawkes.Membership.Profile)` to our the schema. We'll need a migration to complete that task - in terminal run `mix ecto.gen.migration add_user_id_to_profiles` and then add our alter table statement:

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

Next let's add a method that will find or create a changeset for a user profile.  In `Fawkes.Membership` add an alias for profile `alias Fawkes.Membership.Profile`. Now let's add two versions of profile_changeset - each using a different implementation of changeset on Profile.

```
  def profile_changeset(%User{profile: profile}) when not is_nil(profile) do
    Profile.changeset(profile, %{})
  end

  def profile_changeset(user) do
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
    |> cast(attrs, [:first, :last, :slug, :company, :github, :twitter,
                    :description, :image])
    |> cast_attachments(attrs, [:image])
    |> validate_required([:first, :last])
    |> validate_exclusion(:slug, [:edit, :new])
    |> unique_constraint(:slug)
  end
```

### Add profile edit page
(Use `git checkout 7c.profile_edit` to catch up with the class)

First we'll need a profile controller. Create a new file named `lib/fawkes_web/controllers/profile_controller.ex`. For now we'll just implement the Edit and Update actions.

```
defmodule FawkesWeb.ProfileController do
  use FawkesWeb, :controller

  alias Fawkes.Membership

  def edit(conn, _params) do
    render(conn, "edit.html", changeset: Membership.profile_changeset(conn.assigns.current_user))
  end

  def update(conn, %{"profile" => params}) do
    case Membership.update_profile(conn.assigns.current_user, params) do
      {:ok, _} ->
        conn
        |> put_flash(:info, "User updated successfully.")
        |> redirect(to: slot_path(conn, :index))

      {:error, %Ecto.Changeset{} = changeset} ->
        render(conn, "edit.html", changeset: changeset)
    end
  end
end
```

We'll need a ProfileView. Create a new file named `lib/fawkes_web/views/profile_view.ex`:

```
defmodule FawkesWeb.ProfileView do
  use FawkesWeb, :view
end
```

Create a new folder named `profile` in `lib/fawkes_web/templates/`. Add the form template itself (with `multipart: true` set) in an edit.html.eex:

```
<div class="container">
  <h2>Edit User</h2>

  <%= form_for @changeset, membership_profile_path(@conn, :update), [multipart: true], fn f -> %>
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
</div>
```

And add them to our router where we use our ensure_auth pipeline. We'll add this as a singleton so that our url to edit the current_users profile is just "/profile/edit"

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
<div class="container speaker-index">
  <h2 class="subheader">Members</h2>
  <div class="row attendee">
    <%= for profile <- @profiles do %>
        <div class="card col-xs-4">
          <div class="card-body ">
            <img src="<%= Fawkes.Uploader.Image.url({profile.image, profile}) %>" class="rounded-circle">
            <h4><%= link FawkesWeb.SharedView.full_name(profile), to: profile_path(@conn, :show, profile.slug) %></h4>
          </div>
        </div>
    <% end %>
  </div>
</div>
```

and a `show.html.eex`

```
<div class="container speaker-details">
  <div class="card details">
    <div class="card-body">
      <img src="<%= Fawkes.Uploader.Image.url({@user.image, @user}) %>" class="rounded-circle speaker-details__pic">
      <div class="clearfix"></div>
      <div class="speaker__info">
        <h4><%= FawkesWeb.SharedView.full_name(@user) %></h4>
        <div class="speaker__handle">
          <a href="https://github.com/<%= @user.github %>">
            <i class="fab fa-github"></i> @<%= @user.github %>
          </a>
        </div>
        <div class="speaker__handle">
          <a href="https://twitter.com/<%= @user.twitter %>">
            <i class="fab fa-twitter"></i> @<%= @user.twitter %>
          </a>
        </div>
      </div>
    </div>
  </div>

  <hr />
  <h4 class="subheader">About</h4>
  <p><%= @user.description %></p>

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
