---
title: Time zones and Daylight Saving Time
date:   2015-10-28 22:59:00
description: As if Timezones where not enough
---

As we have grown bigger at [TaskRabbit](https://www.taskrabbit.com) in the US and out of the country, we are heavily using time zones to allow Clients to hire Taskers for future dates.

On November 2nd, California went from PDT to PST. This changed the UTC offset from `-7` to `-8` hours.

The following test was breaking. It is testing the `start_datetime_from_date_slot` method: with a date, a slot, and a time zone it will return the time where the slot will begin that day.

```ruby

describe ".start_datetime_from_date_slot" do
  it "returns the start time from a date and a slot" do
    options = {
      date: "2014-07-18",
      slot: :morning,
      time_zone: "Pacific Time (US & Canada)",
    }
    start_datetime = Schedule.start_datetime_from_date_slot(options)

    expect(start_datetime.strftime("%Y-%m-%d %H:%M:%S")).to eql "2014-07-18 08:00:00"
  end
end

```

This test started failing on November 3rd.

```ruby
expect(start_datetime.strftime("%Y-%m-%d %H:%M:%S")).to eql "2014-07-18 08:00:00"
       expected: "2014-07-18 08:00:00"
            got: "2014-07-18 09:00:00"

```

Here is the declaration of the class:

```ruby
require 'active_support/time_with_zone'

class Schedule

  WINDOWS = {
    morning:   [8, 12],
    afternoon: [12, 16],
    night:     [16, 20],
  }

  def self.start_datetime_from_date_slot(date:, slot:, time_zone:)
    start_hour = WINDOWS[slot].first
    tz_abbrev  = ActiveSupport::TimeZone.new(time_zone).now.strftime("%Z")
    Time.parse("#{date} #{start_hour}:00 #{tz_abbrev}").in_time_zone(time_zone)
  end

end
```

The issue is that the time zone is fetched using the current time, and the current time changes the time zone offset.

For example:

- On the 3rd of November 2014:

```ruby
Timecop.freeze(Date.parse("2014-11-03")) do
  ActiveSupport::TimeZone.new(time_zone).now.strftime("%Z") #=> PST
end
```

- On the 1st of November 2014:

```ruby
Timecop.freeze(Date.parse("2014-11-01")) do
  ActiveSupport::TimeZone.new(time_zone).now.strftime("%Z") #=> PDT
end
```

The solution here was simply get the time zone from the date instead of using of being relative to now:

```ruby
def self.start_datetime_from_date_slot(date:, slot:, time_zone:)
  start_hour = WINDOWS[slot].first
  ActiveSupport::TimeZone.new(time_zone).parse("#{date.strftime("%Y-%m-%d")} #{start_hour}:00")
end
```

A similar issue occured in another method that returns time zones for a specified UTC offset:

```ruby
def time_zones_from_utc_offset(utc_offset:)
  ActiveSupport::TimeZone.select {|tz| tz.now.utc_offset.to_i == utc_offset }
end
```

Now, we require the date:

```ruby
def time_zones_from_utc_offset(utc_offset:, date:)
  ActiveSupport::TimeZone.select {|tz| tz.parse(date).utc_offset.to_i == utc_offset }
end
```

So if you are relying on time zones offset, use the **relevant date** as a reference instead of `Time.now`.

Other thoughts:

- Never store `Pacific Daylight Time (PDT)` or `Pacific Standard Time (PST)` in the database, it should be `Pacific Time (US & Canada)`
- Be careful when parsing a date with the timezone:
  - `Time.parse("2014-11-01 12:00 PDT") #=> 2014-11-01 12:00:00 -0700`
  - `Time.parse("2014-11-01 12:00 PST") #=> 2014-11-01 13:00:00 -0700`

*Original post written on the [TaskRabbit Blog](http://tech.taskrabbit.com/blog/2014/11/07/time-zones-and-daylight-saving-time/).*
