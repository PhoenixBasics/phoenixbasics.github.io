---
layout: page
title: Populate the database exercise answers
---

## Exercise #1: Load speakers
(Use git checkout `5b.load_speaker` to catch up with the class)

1. In the `lib/fawkes/schedule/schedule.ex`, find the function `list_profiles`. Change it to `list_speakers`.
2. Change the list speaker to preload talk

      ```
      def list_speakers do
        Speaker
        |> preload(:talk)
        |> Repo.all()
      end
      ```

3. Open `lib/fawkes_web/controllers/speaker_controllers.ex`, update the `index` function to reference speakers instead of profile. The code should look like this:

      ```
        def index(conn, _params) do
          speakers = Schedule.list_speakers()
          render(conn, "index.html", speakers: speakers)
        end
      ```

4. Let's update the layout. Open `lib/fawkes_web/templates/speaker/index.html.eex`

      ```
      <div class="container speaker-index">
        <h4 class="subheader">Speakers</h4>
        <div class="row attendee">
          <%= for speaker <- @speakers do %>
              <div class="card col-xs-4">
                <div class="card-body ">
                  <img src="<%= speaker.image_url %>" class="rounded-circle">
                  <h4><%= link FawkesWeb.SharedView.full_name(speaker), to: speaker_path(@conn, :show, speaker) %></h4>
                  <p><%= link speaker.talk.title, to: "/talks/#{speaker.talk.id}" %></p>
                </div>
              </div>
          <% end %>
        </div>
      </div>
      ```

5. Go to [http://localhost:4000/speakers](http://localhost:4000/speakers), make sure the speaker loads


## Exercise #2: Show talk
(Use git checkout `5c.show_talk` to catch up with the class)

1. In `lib/fawkes/schedule/schedule.ex`, find the function `get_talk`. Preload slot, speakers, categories, audience, location

      ```
      def get_talk!(id) do
        Talk
        |> preload([:slot, :speakers, :categories, :audience, :location])
        |> Repo.get!(id)
      end
      ```

2. Open `lib/fawkes_web/templates/talk/show.html.eex`. Replace the content with this HTML:

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
          <img src="https://lh3.googleusercontent.com/-fEyw5rjNaN0/VIcChZfCYDI/AAAAAAAADfA/OW_u5voliss/w426-h351/good-job-job-lucky-star-konata-izumi-good-anime-otakus-1375040667.jpg" class="rounded-circle">
          <i class="fas fa-check-circle attendee__check"></i>
          <h4>Name name</h4>
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
