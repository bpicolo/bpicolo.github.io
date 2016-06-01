---
layout: post
title:  "Painlessly run Postgresql schemas in Docker for integration testing"
date:   2016-06-01 14:52:52
categories: docker testing
---
Docker is a great way to help make your integration tests extremely consistent
across all environments, whether they're running on your localhost or on a CI
server like Jenkins. Docker can provide a huge benefit in letting you start
with a fresh copy of your database every single time you run tests, even in
cases where things go wrong.

The last thing you want to do to your test environment, however, is make tests slower to run and iterate
on. The longer it takes to run tests, the more likely you are to get just a bit
lazy about what passes for acceptable testing.

When creating my integration test suite, I hit a bit of pain waiting for schema
files to run and create tables prior to running tests when booting up
the Postgresql Docker container. If you have enough schemas to run, indexes, things
of that nature, it can be pretty painful to wait for prior to every test run.

Luckily, Docker has a convenient way to solve this, the [Dockerfile Layer Cache][docker-cache].
Every command run by a Dockerfile is cached into a new layer. If none of the files
relevant to a command have changed (and none of the previous layers have changed)
since your last docker build, it will skip rebuilding that layer.

The trick, then, is to add schemas to the database while building your docker container.
Then, the simple act of restarting your docker container gives you an entirely
fresh copy of your database.

Unfortunately, the default postgres container makes it a bit hard. If you take
a look at the [postgres 9.5 dockerfile][default-dockerfile] you can see that it
does all of it's database initializing logic on container startup, rather than
in the docker commands. Luckily, it's easy to run that set of commands in the
docker build instead. (At least, I haven't found a good reason why not to).

[This github repo][example] provides a simple example for doing just this. If you
use a migration tool for your database, or something similar, it should also be doable,
you'll just have to add a few more dependencies to the postgres container.

If we `docker-compose run postgres bash` and start up postgres. We can see that
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
bug free with a touch of Docker. With a bit of effort it's a please!

[docker-cache]: https://docs.docker.com/engine/userguide/eng-image/dockerfile_best-practices/#build-cache
[example]: https://github.com/bpicolo/postgres-docker-layer-cache-schemas
[default-dockerfile]: https://github.com/docker-library/postgres/blob/04b1d366d51a942b88fff6c62943f92c7c38d9b6/9.5/Dockerfile
