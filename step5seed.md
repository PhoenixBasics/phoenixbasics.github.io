---
layout: page
title: Populate the database
---


Now that we have our modules, let's connect them all together?

Make a direction in `schedule` called `seed`

```
mkdir lib/fawkes/schedule/seed
```

Run this command to download the file to seed

```
curl -o lib/fawkes/schedule/seed/audience.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/audience.ex && curl -o lib/fawkes/schedule/seed/category.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/category.ex && curl -o lib/fawkes/schedule/seed/event.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/event.ex && curl -o lib/fawkes/schedule/seed/location.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/location.ex && curl -o lib/fawkes/schedule/seed/slot.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/slot.ex && curl -o lib/fawkes/schedule/seed/speaker.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/speaker.ex && curl -o lib/fawkes/schedule/seed/talk.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/talk.ex && curl -o lib/fawkes/schedule/seed/seed.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes/schedule/seed/seed.ex
```

Add this line to `priv/repo/seeds.exs` run the seed file:

```
Fawkes.Schedule.Seed.perform()
```

Let's seed our database:

```
mix run priv/repo/seeds.exs
```

### Querying for data

When we load the schedule, we want to load the talk, slot, location, and category. Let's open `lib/fawkes/schedule/schedule.ex`. First thing we want to do add the ability to query. So import in `Ecto.Query` on line 4.

```
import Ecto.Query
```

We're going to load all the talks through the slots. Go to the function `list_schedule_slots` (maybe line 126). We want to say load all the event and talks. For all the talks, list the slot, speakers, categories, audience, and location.

```
Slot
|> preload([:event, talks: [:slot, :speakers, :categories, :audience, :location]])
|> order_by(asc: :start_time)
|> Repo.all()
|> Enum.group_by(&(Timex.beginning_of_day(&1.start_time)))
```

Now let's update our layout

```
curl -o lib/fawkes_web/templates/slot/index.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/slot/index.html.eex && curl -o lib/fawkes_web/templates/speaker/index.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/speaker/index.html.eex && curl -o lib/fawkes_web/templates/speaker/show.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/speaker/show.html.eex && curl -o lib/fawkes_web/views/shared_view.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/views/shared_view.ex
```

Restart your server. Press `Cmd` + `c` to quit. Then run:

```
mix phx.server
```


### Exercise: Load speakers
1. In the `lib/fawkes/schedule/schedule.ex`, update the function `list_speakers`
2. It should preload `talk`
3. Go to [http://localhost:4000/speakers](http://localhost:4000/speakers), make sure the speaker loads

### Exercise #2: Show talk

Lab Add path for talk

1. Add a route to show talk `/talks/:id`
2. Add a controler to handle the route
3. In Schedule, retrieve the talk with a given id. Preload slot, ,speakers, ,categories, ,audience, ,location
4. Add a view for the talk
5. Add a template for the view. Here's the HTML for it.

```
<div class="container talk">
  <div class="card section">
    <p class="subtext"><%= Timex.format!(@talk.slot.start_time, "%-I:%M %p", :strftime)%> - <%= Timex.format!(@talk.slot.end_time, "%-I:%M %p", :strftime) %></p>
    <h2><%= @talk.title %></h2>
    <p class="smalltext"><strong>Location</strong>: <%= @talk.location.name %></p>
    <p class="icons">
      <button class="btn talk__audience"><%= @talk.audience.name %></button>
      <%= for category <- @talk.categories do %>
        <%= if category.icon_url do %>
          <img src="<%= category.icon_url %>">
        <% else %>
        <button class="btn talk__category">
          <%= category.name %>
        </button>
        <% end %>

      <% end %>
    </p>
    <div class="add-to-schedule action">
      <i class="far fa-calendar-check"></i>
      ADD TO AGENDA
    </div>
  </div>

  <hr/>
  <h4 class="subheader">Speakers/Trainers</h4>
  <%= for speaker <- @talk.speakers do %>
    <div class="card speaker">
      <div class="row card-body">
        <div class="col-sm-3">
          <img src="<%= speaker.image_url %>" class="rounded-circle">
        </div>
        <div class="col-sm-7 speaker__info">
          <a href="/speakers/<%= speaker.id %>"<h4><%= FawkesWeb.SharedView.full_name(speaker) %></h4></a>
          <div class="speaker__handle">
            <i class="fab fa-github"></i> @<%= speaker.github %>
          </div>
          <div class="speaker__handle">
            <i class="fab fa-twitter"></i> @<%= speaker.twitter %>
          </div>
        </div>
        <div class="col-sm-1 speaker__link">
          <a href="/speakers/<%= speaker.id%>"><i class="fas fa-chevron-right"></i></a>
        </div>
      </div>
    </div>
  <% end %>
  <hr />
  <h4 class="subheader">About the event</h4>
  <p><%= raw(@talk.description) %></p>

  <hr />
  <h4 class="subheader">Checked in attendees</h4>
  <div class="row attendee">
    <%= for _ <- (1..10) do %>
      <div class="card col-sm-3">
        <div class="card-body">
          <img src="https://elixirconf.com/2018/images/speakers/boyd-multerer.jpg" class="rounded-circle">
          <i class="fas fa-check-circle attendee__check"></i>
          <h4>Sandra Bullock</h4>
          <div class="speaker__handle">
            <i class="fab fa-github"></i> @TODO-GIthubhandle
          </div>
          <div class="speaker__handle">
            <i class="fab fa-twitter"></i> @TODO-twitterhandle
          </div>
        </div>
      </div>
    <% end %>
  </div>
</div>
```

Go to [http://localhost:4000/talks/1](http://localhost:4000/talks/1)
It should work. 
