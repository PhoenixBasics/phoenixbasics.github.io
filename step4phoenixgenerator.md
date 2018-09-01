---
layout: page
title: Phoenix Generator
---

(Use `git checkout 4.phoenix_generator` to catch up with the class)

Because adding CRUD application is so common, Phoenix comes with a task to generate all that code for us. Let's generate a model to represent the level of the talk (e.g. beginner, advance).

```
mix phx.gen.html Schedule Audience audiences slug:string:unique name:string
```

Notice at the end of the command, it told us to add the new route `resources "/audiences", AudienceController` to our router. This will map all our CRUD operations. Let's open our router `lib/fawkes_web/router.ex` and add:

```
resources "/audiences", AudienceController
```

To check to see what routes were added, let's run

```
mix phx.routes
```

Now let's run our migration to create the table:

```
mix ecto.migrate
```

Alright, let's check to see if it works [http://localhost:4000/audiences](http://localhost:4000/audiences).

Well, that was a lot of work.

## Other models
(Use `git checkout 4a.generate_other_models` to catch up with the class)

Let's generate a few more models for our application.

```
mix phx.gen.html Schedule Slot schedule_slots slug:string:unique start_time:utc_datetime end_time:utc_datetime
```

Let's add the resource to our router `lib/fawkes_web/router.ex`:

```
resources "/schedule_slots", SlotController
```

Now let's add locations. Because we don't need the front end for this, we will only generate the schema:

```
mix phx.gen.schema Schedule.Location locations slug:string:unique name:string
```

Same for event. 

```
mix phx.gen.schema Schedule.Event events slug:string:unique name:string slot_id:references:schedule_slots
```

For the talk, we need to view the talks, so we will generate everything.

```
mix phx.gen.html Schedule Talk talks slug:string:unique title:string slot_id:references:schedule_slots audience_id:references:audiences location_id:references:locations description:text
```

Let's add the resource to our router `lib/fawkes_web/router.ex`:

```
resources "/talks", TalkController
```

## Speaker
(Use `git checkout 4b.speaker` to catch up with the class)

Before we generate the speaker, let's generate a migration to add citext to our database. Citext will ignore case when searching for text.

```
mix ecto.gen.migration enable_citext_extension
```

Open up the migration file `priv/repo/migrations/[date]_enable_citext_extension.exs`. Add code to add citext.

```
defmodule Fawkes.Repo.Migrations.EnableCitextExtension do
  use Ecto.Migration

  def change do
    execute "CREATE EXTENSION citext", "DROP EXTENSION citext"
  end
end
```

Now we're ready to generate the speaker. 

```
mix phx.gen.html Schedule Speaker profiles talk_id:references:talks slug:string:unique image:string image_url:string first:string last:string company:string github:string twitter:string description:text
```

Let's add the routes for the speakers:

```
resources "/speakers", SpeakerController
```

We need one more migration to link the category to the talks:

```
mix ecto.gen.migration create_talks_categories
```

Open up the migration file `priv/repo/migrations/[date]_create_talks_categories.exs`, add the code for the migration

```
defmodule Fawkes.Repo.Migrations.CreateTalksCategories do
  use Ecto.Migration

  def change do
    create table(:talks_categories) do
      add :talk_id, references(:talks, on_delete: :nothing)
      add :category_id, references(:categories, on_delete: :nothing)
    end

    create index(:talks_categories, [:talk_id])
    create index(:talks_categories, [:category_id])
  end
end
```

Now let's run our migration to add all the tables:

```
mix ecto.migrate
```

## Relationships
(Use `git checkout 4c.relationships` to catch up with the class)

Now that we have our models generated, let's define their relationship. Open the `Category` module `lib/fawkes/schedule/category.ex` and add this code in the schema, below the fields to say that a `Category` has many talks.

```
many_to_many :talks, Fawkes.Schedule.Talk, join_through: "talks_categories"
```

For `Audience` module `lib/fawkes/schedule/audience.ex`, add add this code in the schema, below the fields to say that an audience has many talks.

```
has_many :talks, Fawkes.Schedule.Talk
```

A speaker belongs to a talk, so open `lib/fawkes/schedule/speaker.ex`, delete the line for the `talk_id` field and replace it with the belongs_to relationship:

```
belongs_to :talk, Fawkes.Schedule.Talk
```

Remove image, company, github, twitter, and description from the required fields.

```
|> validate_required([:slug, :image_url, :first, :last])
```

Add `talk_id` to the cast

```
|> cast(attrs, [:slug, :image, :image_url, :first, :last, :company, :github, :twitter, :description, :talk_id])
```

Open `lib/fawkes/schedule/talk.ex`, <span style="color:red">delete the fields for audience, location, slot</span>. Add the following relationships:

```
many_to_many :categories, Fawkes.Schedule.Category, join_through: "talks_categories"
belongs_to :audience, Fawkes.Schedule.Audience
belongs_to :location, Fawkes.Schedule.Location
belongs_to :slot, Fawkes.Schedule.Slot
has_many :speakers, Fawkes.Schedule.Speaker
```

So that your schema section looks like this:

```
  schema "talks" do
    field :description, :string
    field :slug, :string
    field :title, :string

    many_to_many :categories, Fawkes.Schedule.Category, join_through: "talks_categories"
    belongs_to :audience, Fawkes.Schedule.Audience
    belongs_to :location, Fawkes.Schedule.Location
    belongs_to :slot, Fawkes.Schedule.Slot
    has_many :speakers, Fawkes.Schedule.Speaker

    timestamps()
  end
```

For the `cast`, add in `:audience_id, :location_id, :slot_id` like this:

```
|> cast(attrs, [:slug, :title, :description, :audience_id, :location_id, :slot_id])
```

In addition to the validation, in the changeset you can set the constraint that the child can't be created if the parent doesn't exist.

```
|> assoc_constraint(:audience)
|> assoc_constraint(:slot)
|> assoc_constraint(:location)
```

Your talk changeset should look like this:

```
def changeset(talk, attrs) do
  talk
  |> cast(attrs, [:slug, :title, :description, :audience_id, :location_id, :slot_id])
  |> validate_required([:slug, :title, :description])
  |> unique_constraint(:slug)
  |> assoc_constraint(:audience)
  |> assoc_constraint(:slot)
  |> assoc_constraint(:location)
end
```

Open `lib/fawkes/schedule/slot.ex`, let's add the talk and event relationship.

```
has_many :talks, Fawkes.Schedule.Talk
has_one :event, Fawkes.Schedule.Event
```

Open `lib/fawkes/schedule/event.ex`, add `slot_id` to the cast call.

```
|> cast(attrs, [:slug, :name, :slot_id])
```

### Add Timex dependency
(Use `git checkout 4d.timex` to catch up with the class)

Open `mix.exs`, after line 33 in the `deps` block, add a comma, then timex_ecto

```
{:timex_ecto, "~> 3.3"}
```

On line 23, add `:timex_ecto` to the extra_applications list as another application to run.

```
extra_applications: [:logger, :runtime_tools, :timex_ecto]
```

Let's get the dependency by running:

```
mix deps.get
```

Now open `lib/fawkes/schedule/slot.ex` again. Change the start and end time to `Timex.Ecto.DateTime` so your slot schema looks like this:

```
schema "schedule_slots" do
  field :end_time, Timex.Ecto.DateTime
  field :slug, :string
  field :start_time, Timex.Ecto.DateTime

  has_many :talks, Fawkes.Schedule.Talk
  has_one :event, Fawkes.Schedule.Event

  timestamps()
end
```

One last thing before we can test it out. Open `lib/fawkes_web/views/slot_view.ex`. Add a function to get the dates.

```
def conference_dates do
  format = "{YYYY}-{M}-{D}"

  [Timex.parse!("2018-09-04", format),
   Timex.parse!("2018-09-05", format),
   Timex.parse!("2018-09-06", format),
   Timex.parse!("2018-09-07", format)]
end
```
