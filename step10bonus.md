---
layout: page
title: Bonus exercise
---

Need more practice? Here are a few more things you can implement to make this app better.

### Speaker details
When go to a speaker page, it throws an error. Note the error says talk is not loaded.

```
key :id not found in: #Ecto.Association.NotLoaded<association :talk is not loaded>
```

In order to make this work, in `lib/fawkes/schedule/schedule.ex`, edit the `get_speaker!(id)` function to preload talk, and the talk's slot, speakers, categories, audience, location. (Hint: check out the `list_schedule_slots` function if you need help.)

### Agenda by day

The agenda by day link works. If you click on a day, it goes to `/schedule_slots`
