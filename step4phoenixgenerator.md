---
layout: page
title: Phoenix Generator
---


Because adding CRUD application is so common and tedious, Phoenix comes with a command to generate all that code for us, and more. Let's generate a model to represent the level of the talk (e.g. beginner).

```
mix phx.gen.html Schedule Audience audiences slug:string:unique name:string
```

Notice at the end of the command, it told us to add the new route `resources "/audiences", AudienceController` to our router. This will map all our CRUD operations. Let's open our router `lib/fawkes_web/router.ex` and add:

```
resources "/audiences", AudienceController
```

#TODO add code to see all the routes

Now let's run our migration to create the table:

```
mix ecto.migrate
```

Alright, let's check to see if it works http://localhost:4000/audiences.

<img src="https://media.giphy.com/media/3o6MbeDiaHJaF2EuQM/giphy.gif">

Woo, that was hard work.

Let's generate a few more models for our application.

```
mix phx.gen.html Schedule Slot schedule_slots slug:string:unique start_time:utc_datetime end_time:utc_datetime
```

Let's add the route to our url

```
resources "/schedule_slots", SlotController
```

```
mix phx.gen.schema Schedule.Location locations slug:string:unique name:string
```

```
mix phx.gen.schema Schedule.Event events slug:string:unique name:string slot_id:references:schedule_slots
```

```
mix phx.gen.schema Schedule.Talk talks slug:string:unique title:string slot_id:references:schedule_slots audience_id:references:audiences location_id:references:locations description:text
```

```
mix phx.gen.html Schedule Speaker speakers talk_id:references:talks slug:string:unique image_url:string first_name:string last_name:string company:string github:string twitter:string description:text
```

Let's add the routes for the speakers:

```
resources "/speakers", SpeakerController
```

Open up `lib/fawkes/schedule/speaker.ex`, on line 25, remove company, github, twitter, and description from the required fields.

```
|> validate_required([:slug, :image_url, :first_name, :last_name])
```


We need one more migration to link the category to the talks:

```
mix ecto.gen.migration add_talks_categories
```

Open up the migration file `priv/repo/migrations/___add_talks_categories.exs`, add the code for the migration

```
defmodule Fawkes.Repo.Migrations.AddTalksCategories do
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


Now that we have our models generated, let's define their relationship. Open the `Category` module `lib/fawkes/schedule/category.ex` and add this code on line `10` to say that a `Category` has many talks.

```
many_to_many :talks, Fawkes.Schedule.Talk, join_through: "talks_categories"
```

For `Audience` module `lib/fawkes/schedule/audience.ex`,

```
has_many :talks, Fawkes.Schedule.Talk
```

A speaker belongs to a talk, so open `lib/fawkes/schedule/speaker.ex`, delete line 15 for the `talk_id` field and replace it with the belongs_to relationship:

```
belongs_to :talk, Fawkes.Schedule.Talk
```

Open `lib/fawkes/schedule/talk.ex`, delete the fields for audience, location, slot. Add the following relationships:

```
many_to_many :categories, Fawkes.Schedule.Category, join_through: "talks_categories"
belongs_to :audience, Fawkes.Schedule.Audience
belongs_to :location, Fawkes.Schedule.Location
belongs_to :slot, Fawkes.Schedule.Slot
has_many :speakers, Fawkes.Schedule.Speaker
```

In addition to the validation, in the changeset you can set the constraint that the child can't be created if the parent doesn't exist.

```
|> assoc_constraint(:audience)
|> assoc_constraint(:slot)
|> assoc_constraint(:location)
```

Open `lib/fawkes/schedule/slot.ex`, let's add the talk and event relationship.

```
has_many :talks, Fawkes.Schedule.Talk
has_one :event, Fawkes.Schedule.Event
```

### Add Timex dependency

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
