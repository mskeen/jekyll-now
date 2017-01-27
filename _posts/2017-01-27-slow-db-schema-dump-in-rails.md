---
layout: post
title: Slow Rails Schema Dump after Mysql Upgrade on macOS Sierra
---

This post might be a bit dry, but hopefully it will be helpful for someone
who runs into the same problem.

### The Problem

In my development environment, I recently noticed a big slowdown when running
database migrations for a Rails application.

The database changes were applied quickly, but then there was a two or three
minute delay before the schema file was updated and the command completed.

At first I thought perhaps I'd messed up something with my custom rake tasks,
but running a migration for the same project on another laptop completed quickly.

To debug, I started by running db:migrate with the --trace option.  This showed
that the task was actually slowing down upon the execution of the
db:schema:dump, and watching the schema.rb file itself.  I could see that the
tables were updated quickly, but that it was a couple of minutes before the
indexes were updated.

The next step was to view the active queries in MySQL to see if there was
something helpful there:

{% highlight sql %}
show full processlist
{% endhighlight %}

It was pretty easy to spot the following query, which was taking over 10
seconds to run and was being execute once for each table in my schema:

{% highlight sql %}
SELECT fk.referenced_table_name AS 'to_table',
       fk.referenced_column_name AS 'primary_key',
       fk.column_name AS 'column',
       fk.constraint_name AS 'name',
       rc.update_rule AS 'on_update',
       rc.delete_rule AS 'on_delete'
FROM information_schema.key_column_usage fk
JOIN information_schema.referential_constraints rc
USING (constraint_schema, constraint_name)
WHERE fk.referenced_column_name IS NOT NULL
  AND fk.table_schema = 'freelancer_development'
  AND fk.table_name = 'event_trackers'
{% endhighlight %}


### The Fix

I started by checking to see which version of MySQL I was running.  I couldn't
remember upgrading in quite some time, so I feared I might be very outdated.  
To my surprise, it was 5.7.16, which is very recent.

As it turns out, I had recently run the following to update all brew packages:

{% highlight bash %}
brew update && brew upgrade
{% endhighlight %}

This, of course, will update all my brew packages, including MySQL.  Perhaps
there was an error at the time, or I simply neglected to run the following, which
was required to complete the update:

{% highlight bash %}
  mysql_upgrade -u root --force
{% endhighlight %}

Once I did this and restarted MySQL (brew services restart mysql), my Rails
migrations dropped from two+ minutes to about eight seconds.
