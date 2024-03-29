A scheduling library (a general-purpose do-er of things), inspired by my favourite features from:

overtone.at-at (https://github.com/overtone/at-at)

+Tasks are added to pools, allowing them to be interrogated at any time

-Originated in music programming; based around millisecond delays (dates/times only via other libraries)

chime (https://github.com/jarohen/chime)

+A nice DSL for scheduling; uses core.async in a way that makes sense to me

-When tasks are created they only leave behind their stop channel - no way to check their state

-Defaults to UTC rather than local time, which has caught me out on several occasions...

I've thrown my hat into the ring for java-time, and built a small chime-inspired DSL around it. If you want to use any of the more advanced java-time predicates, just require it and use them as they come (see below).

---

USAGE

```
[org.clojars.tafenton/do-er "1.0.4"]

(use 'do-er.core)
```

First off, create your task pool:

```clojure
(def mypool (make-task-pool))     ; Plot twist - it's just an empty map in an atom
```

Then start building your tasks. Successfully-created tasks will return their ids (a keywordized UUID if none is provided).

```clojure
(add-task {:id :hello-forever
           :task-pool mypool
           :function #(println "Hello, world")
           :schedule (every 5 :seconds)})
=> :hello-forever
Hello, world
Hello, world
Hello, world
...
```

This will rapidly become annoying, so:

```clojure
(stop-task mypool :hello-forever)
=> true
```

Note this only stops future triggers, and does not stop any scheduled functions which are currently running. Stopping a stopped task will return nil.

To interrogate your schedules just deref the task pool:

```clojure
(clojure.pprint/pprint @mypool)
=> {:hello-forever
     {:run-count 3
      :next-run nil
      :state :stopped
      :last-completed [java.time.LocalDateTime ... "2023-09-13T10:54:19.147608900"]
      :stop-chan [clojure.core.async.impl.channels.ManyToManyChannel ...]}}
```

Of course, if the task was still running then :next-run would be another java.time.LocalDateTime.

---

Tasks that would never execute, i.e. because their entire schedules are in the past, will return false on creation and won't be added to the task pool:

```clojure
(add-task {:id :never-gonna-happen
           :task-pool mypool
           :function #(println "You won't see this")
           :schedule (-> (every 5 :minutes)
                         (from (java-time/local-date-time 2019 1 1))
                         (until (java-time/local-date-time 2019 2 1)))})
=> false
```

Note the interaction with java-time. You can also specify tasks that run on completion:

```clojure
(add-task {:id :send-help
           :task-pool mypool
           :function #(println "Help!")
           :schedule (-> (every 5 :seconds)
                         (until (in 15 :seconds)))
           :on-complete #(println "Finally, I'm free!")})
=> :send-help
Help!
Help!
Help!
Finally, I'm free!
```

And tasks that run on error, taking the caught exception as an argument (note that do-er will continue to run the function as scheduled, even if it errors every time):

```clojure
(add-task {:id :doomed-to-fail
           :task-pool mypool
           :function #(println (/ 1 0))
           :schedule (once (in 2 :seconds))
           :on-error (fn [e] (timbre/error     ; Other logging libraries are available
                               (-> e                
                                   Throwable->map
                                   :cause)))})
=> :doomed-to-fail
19-09-14 13:43:44 MACHINE-NAME ERROR [do-er.core:5] - Divide by zero
```

Other examples with the scheduling DSL:

```clojure
(add-task {:id :hello-briefly
           :task-pool mypool
           :function #(println "Hello, wo...")
           :schedule (-> (every 5 :seconds)
                         (limit 1))})
```

```clojure
(add-task {:id :weekend-worker
           :task-pool mypool
           :function #(println "I need a vacation")
           :schedule (-> (every 15 :minutes)
                         (only saturdays sundays)})
```

---

Improvements that aren't necessary for my use case, but that I might get round to one day:

If you schedule a task with a non-specific schedule and no start time, i.e. every 60 seconds effective immediately, it may run immediately OR it may wait for the next scheduled execution.

When a task finishes running, it will discard any triggers that have elapsed; e.g. if a task starts hourly but takes 65 minutes to complete, it will wait 55 minutes before starting again. For me this is preferable, but I may add the ability to vary this.

Schedules are not durable - my use case is small self-contained data-load applications, so I have the luxury of hard-coding the schedule and not having to worry about file system permissions etc.
