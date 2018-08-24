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
Fawkes.Schedule.seed()
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
curl -o assets/css/zapp.css https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/assets/css/zapp.css && curl -o lib/fawkes_web/templates/slot/index.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/slot/index.html.eex && curl -o lib/fawkes_web/templates/speaker/index.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/speaker/index.html.eex && curl -o lib/fawkes_web/templates/speaker/show.html.eex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/templates/speaker/show.html.eex && curl -o lib/fawkes_web/views/shared_view.ex https://raw.githubusercontent.com/nhu313/fawkes_nhu/master/lib/fawkes_web/views/shared_view.ex
```



### Lab Add path for talk


```
<div class="container talk">
  <div class="card section">
    <p class="subtext"><%= format_date(@talk.slot.start_time) %> <%= format_time(@talk.slot.start_time) %> - <%= format_time(@talk.slot.end_time) %></p>
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
          <a href="/speakers/<%= speaker.id %>"<h4><%= full_name(speaker) %></h4></a>
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
