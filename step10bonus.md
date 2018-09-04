---
layout: page
title: Bonus exercise
---

Need more practice? Here are a few more things you can implement to make this app better.

## Attendees

(Use `git checkout 10.bonus1` to catch up with the class)

Open up `lib/fawkes_web/templates/talk/show.html.eex`. Scroll down to line 55 where it shows the list of attendees. Note that all these attendees are fake. In this exercise, you will retreive the users who added this talk to their agendas.

1. Open up `lib/fawkes/agenda/agenda.ex`. Add a function to retrieve the user given a talk id. Preload the user profile.
2. Open up `lib/fawkes_web/controller/talk_controller.ex`, in the `def show(conn, %{"id" => id})` function, call agenda to retrieve the users. Add that to the render function.
3. Open up `lib/fawkes_web/templates/talk/show.html.eex`, change the for loop to get the user `<%= for user <- @users do %>`. Add a wrapper around the image to only show it if there is a profile. Then change the hardcode image to the user image. Add a wrapper around the github and twitter handle to only show it if there is a profile. Here is the result from the for loop.

```
<%= for user <- @users do %>
      <div class="card col-sm-3">
        <div class="card-body">
          <%= if user.profile do %>
            <img src="<%= Fawkes.Uploader.Image.url({user.profile.image, user.profile}) %>" class="rounded-circle">
          <% end %>
          <i class="fas fa-check-circle attendee__check"></i>
          <h4><%= user.username %></h4>
          <%= if user.profile do %>
            <div class="speaker__handle">
              <i class="fab fa-github"></i> <%= user.profile.github %>
            </div>
            <div class="speaker__handle">
              <i class="fab fa-twitter"></i> <%= user.profile.twitter %>
            </div>
          <% end %>
        </div>
      </div>
    <% end %>
```

See the answer on [Github](https://github.com/PhoenixBasics/fawkes/compare/10.bonus1...10.bonus2?expand=1)

## Agenda by day
(Use `git checkout 10.bonus2` to catch up with the class)

In this exercise we'll filter the talks by the day. Notice if you click on any of the date, it hits `schedule_slots` with the date param ([http://localhost:4000/schedule_slots?date=2018-09-07](http://localhost:4000/schedule_slots?date=2018-09-07)).

1. Open `lib/fawkes_web/controller/slot_controller.ex`, edit the `index` function to pass the params date to `Schedule.list_schedule_slots`. Open up `lib/fawkes/schedule/schedule.ex`, change the `list_schedule_slots` function to take in a date. We want to parse the date, then get the beginning of the date and end of the date. Then filter to only return the slot within the given time frame.

```
parsed_date = Timex.parse!(date, "%Y-%m-%d", :strftime)
beginning_of_day = Timex.beginning_of_day(parsed_date)
end_of_day = Timex.end_of_day(parsed_date)
```
