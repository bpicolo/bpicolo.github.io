---
layout: post
title:  "Painlessly run Postgresql schemas in Docker for integration testing"
date:   2016-06-01 14:52:52
categories: docker testing
comments: true
---

Docker is a great way to help make your integration tests consistent
across all environments, whether they be on your localhost or on a CI
server (e.g. Jenkins). It lets you start with a fresh copy of your database
every single time you run tests, even when things go wrong. This is a huge benefit
because it allows you to entirely eliminate database test pollution between runs of your
test suite. This post will describe how to speed up that workflow.

The last thing anyone wants to do to their test environment is make it slower, and,
let's be honest - the longer it takes to run and iterate on tests, the more likely you are to get lazy
about what passes for acceptable testing.

When creating my integration test suite, I was really annoyed that I had to wait for schema
files to run and create tables each time I started a test run. With enough schemas and indexes (or other commands) to run,
the wait before every test run can become pretty painful.

Luckily, Docker has a convenient way to solve this: the [Docker Layer Cache][docker-cache].
Every command run by a Dockerfile is cached into a new 'layer'. If none of the files
relevant to a command have changed (and none of the previous layers have changed)
since your last docker build, it will skip rebuilding that layer.

The trick, then, is to add schemas to the database while building your docker container.
After that, just restarting your docker container gives you an entirely
fresh copy of your database.

However, this process isn't as straightforward as it should be.
If you take a look at the [postgres 9.5 dockerfile][default-dockerfile] you can see that it
does all of its database initializing logic on container startup, rather than
in the docker commands. Luckily, it's easy to run that set of commands in the
docker build instead. I haven't found a good reason not to do this yet -- it's actively
beneficial.

[This github repo][example] provides a simple example for doing this. Even if you
use a migration tool for your database (or something similar) it should be doable --
you'll just have to add a few more dependencies to the postgres container.

The important parts:
* We have our schemas enumerated in separated SQL files for running
* We copied most of the [default entrypoint][[default-docker-init] to the postgres 9.5
container, put it in docker-init, and now run it as part of the docker build, rather
than as an entrypoint.
* We added a command to run all of the SQL files enumerated in the other directory, [add_schemas.sh][add-schemas].

If we `docker-compose run postgres bash` and start up postgres, we can see that
our schemas are ready for us.

{% highlight bash %}
root@93cc994e41a0:/# psql -U postgres
psql (9.5.3)
Type "help" for help.

postgres=# ls
postgres-# \d
                List of relations
 Schema |       Name        |   Type   |  Owner
--------+-------------------+----------+----------
 public | user              | table    | postgres
 public | user_id_seq       | sequence | postgres
 public | user_tacos        | table    | postgres
 public | user_tacos_id_seq | sequence | postgres
(4 rows)
{% endhighlight %}

That's all there is to it! Make your integration testing workflow faster and
more bug free with a touch of Docker. Takes a bit of effort, but at the end it makes
for a really great developer experience.

[docker-cache]: https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#build-cache
[example]: https://github.com/bpicolo/postgres-docker-layer-cache-schemas
[default-dockerfile]: https://github.com/docker-library/postgres/blob/04b1d366d51a942b88fff6c62943f92c7c38d9b6/9.5/Dockerfile#L56
[default-docker-init]: https://github.com/docker-library/postgres/blob/04b1d366d51a942b88fff6c62943f92c7c38d9b6/9.5/docker-entrypoint.sh
[add-schemas]: https://github.com/bpicolo/postgres-docker-layer-cache-schemas/blob/master/docker-entrypoint-initdb.d/add_schemas.sh
