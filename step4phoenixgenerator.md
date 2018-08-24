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

Now let's run our migration to create the table:

```
mix ecto.migrate
```

Alright, let's check to see if it works http://localhost:4000/audiences.

<img src="https://media.giphy.com/media/3o6MbeDiaHJaF2EuQM/giphy.gif">

Let's generate a few more models for our application.

First we need the slot

```
mix phx.gen.html Schedule Slot schedule_slots slug:string:unique start_time:utc_datetime end_time:utc_datetime
```

```
mix phx.gen.html Schedule Location locations slug:string:unique name:string
```

```
mix phx.gen.html Schedule Event events slug:string:unique name:string slot_id:references:schedule_slots
```

```
mix phx.gen.html Schedule Talk talks slug:string:unique title:string slot_id:references:schedule_slots audience_id:references:audiences location_id:references:locations description:text
```

```
mix phx.gen.html Schedule Speaker speakers talk_id:references:talks slug:string:unique image_url:string first_name:string last_name:string company:string github:string twitter:string description:text
```

Now that we have our models generated, let's define their relationship. Open the `Category` module `lib/fawkes/schedule/category.ex` and add this code on line `10` to say that a `Category` has many talks.

```
many_to_many :talks, Fawkes.Schedule.Talk, join_through: "talks_categories"
```

For `Audience` module `lib/fawkes/schedule/audience.ex`,

```
has_many :talks, Fawkes.Schedule.Talk
```

Open `speaker.ex`, delete line 15 and replace it with the belongs_to relationship:

```
belongs_to :talk, Fawkes.Schedule.Talk
```

Open `talk.ex`, delete the fields for audience, location, slot. Add the following relationships:

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
```



Notice at the end of the command, it told us to add the new route `resources "/speakers", SpeakerController` to our router. This will handle all of our CRUD operations. Let's open our router `lib/fawkes_web/router.ex` and add:

```
resources "/speakers", SpeakerController
```

Let's run the migration to create our speaker table

```
mix ecto.migrate
```

Alright, now let's restart our server and check http://localhost:4000/speakers.

Woo, that was hard work.

Let's add a few more.





Now we generated all our modules, let's add them to the router `lib/fawkes_web/router.ex` (if you haven't already). This should go right below `resources "/speakers", SpeakerController`

```
resources "/schedule_slots", SlotController
resources "/locations", LocationController
resources "/talks", TalkController
resources "/events", EventController
resources "/speakers", SpeakerController
```

Now let's run our migration to add all the tables:

```
mix ecto.migrate
```


mix phx.gen.html MyContext MyModel my_models a b c d:integer --no-context

--no-model
https://hexdocs.pm/phoenix/Mix.Tasks.Phoenix.Gen.Html.html
mix phoenix.gen.model Post posts title user_id:references:users


mix phoenix.gen.model TalkCategory talks_categories talk_id:references:talks category_id:references:categories --no-model

mix ecto.gen.migration add_talk_category


defmodule Fawkes.Repo.Migrations.CreateTalkCategory do
  use Ecto.Migration

  def change do
    create table(:talks_categories) do
      add :talk_id, references(:talks, on_delete: :nothing)
      add :category_id, references(:categories, on_delete: :nothing)

      timestamps()
    end

    create index(:talks_categories, [:talk_id])
    create index(:talks_categories, [:category_id])
  end
end


https://medium.com/@abitdodgy/building-many-to-many-associations-with-embedded-schemas-in-ecto-and-phoenix-e420abc4c6ea
