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

Woo, that was hard work.

Let's generate a few more models for our application.

```
mix phx.gen.html Schedule Slot schedule_slots slug:string:unique start_time:utc_datetime end_time:utc_datetime
```

```
mix phx.gen.model Schedule Location locations slug:string:unique name:string
```

```
mix phx.gen.model Schedule Event events slug:string:unique name:string slot_id:references:schedule_slots
```

```
mix phx.gen.model Schedule Talk talks slug:string:unique title:string slot_id:references:schedule_slots audience_id:references:audiences location_id:references:locations description:text
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

A speaker belongs to a talk, so open `speaker.ex`, delete line 15 and replace it with the belongs_to relationship:

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
|> assoc_constraint(:slot)
|> assoc_constraint(:location)
```

Now we generated all our modules, let's add them to the router `lib/fawkes_web/router.ex` (if you haven't already). This should go right below `resources "/audiences", AudienceController`

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
