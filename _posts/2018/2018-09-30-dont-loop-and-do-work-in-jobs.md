---
title: Don’t Loop & Do Work in Jobs
description: 'Loop and enqueue, then work on one object.'
layout: article
category: ruby
image:
  base: '2018/dont-loop-and-do-work-in-jobs'
  alt: 'Fruit Loops'
  source: 'https://unsplash.com/photos/chp1ITgplkA'
  credit: 'Etienne Girardet'
---

Getting as much of the slow or non-essential work of your application into asynchronous jobs is a good idea for the overall performance of your application.

## Instead of…

…doing a single job that iterates over a group of objects and does some work on each one:

```ruby
class DoABunchOfTranslationsJob < ApplicationJob
  def perform
    Text.find_each do |text|
      text.do_a_slow_translation
    end
  end
end
```


## Use…

…an initial ‘enqueuing’ job to create many small, independent jobs that act on each object individually.

```ruby
class DoABunchOfTranslationsJob < ApplicationJob
  Text.find_each do |text|
    DoASingleTranslationJob.perform_later(text)
  end
end

class DoASingleTranslationJob < ApplicationJob
  def perform(text)
    text.do_a_slow_translation
  end
end
```


## But why?

Jobs should ideally run as quickly as possible and make use of the concurrency of your background workers.

Jobs can fail for multiple reasons: errors raised within the job itself or errors _external_ to the job related to the environment (e.g. a reboot, some sort of system error etc.).

If your job is long-running the chance of the job being interrupted 'mid flow' increases. You might also see large memory usage in such tasks.

If a long-running job encounters an error, you have two problems: the work it is doing is left in an inconsistent state and the long-running job will have to be run again from the beginning. This means some work will be repeated, perhaps multiple times if the job fails. It might, in some circumstances, never finish.

By breaking the work down into tiny repeatable pieces you increase the resilience of both the individual jobs and the wider ‘task’ as a whole.

This style also has the benefit of making your code finish more quickly. With lots of small 'chunks' you can make use of concurrency and run many jobs at the same time, rather than running all the activity sequentially.


### Why not?

There's an extra level of indirection; you end up creating a `BulkWhatever` job for every task to enqueue all the `Whatever` jobs. This means the code is more complex and will perhaps be more confusing when you come back to it later.

For short-running loops this extra complexity might be overkill.
