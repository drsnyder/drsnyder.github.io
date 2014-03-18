---
layout: post
title: Seek to a position in an ordered relation using PostgreSQL
---

I need to seek to a given position within a sorted relation to fetch an object
and it's neighbors. To get the sorted relation I also need to filter by
categories and a status column and then assign a position to each tuple in
the relation so that I can seek to the position that I want.

Here is an example. The where clause specifies the position and how many
neighbors are needed.

{% highlight sql %}
WITH object_positions AS (
  SELECT object_container_map.container_id, objects.id, containers.category_id,
   row_number() OVER (PARTITION BY object_container_map.container_id
                      ORDER BY objects.id DESC) AS index
  FROM objects
  JOIN object_container_map ON (
    objects.id = object_container_map.object_id
    AND object_container_map.is_default = false
  )
  JOIN object_container_map default_map ON (
    objects.id = default_map.object_id
    AND default_map.is_default = true
  )
  JOIN containers ON (
    containers.id = default_map.container_id
  )
  WHERE objects.status = 1 
    AND containers.status = 1 
    AND object_container_map.container_id = 500001
  AND containers.category_id IN (2,3,4,5,6)
  ORDER BY objects.id DESC

)
SELECT row_number() OVER (), container_id, id AS object_id, category_id, index
FROM object_positions
WHERE index BETWEEN 50000 - 6 AND 50000 + 6 
ORDER BY index ASC;
{% endhighlight %}

The problem with is that with 2M+ rows in the objects table this query takes
about 3000ms on a modern laptop and about 900ms on production database hardware.
Even at 900ms that's an order of magnitude (maybe 2) longer than most high
traffic websites can tolerate.

Because of the filtering and the need to find the object at a given position you
need to materialize the sorted and filtered relation somehow before you seek to
the position that you want. That's what the CTE is doing. I upload an explain
from sample data on [explain.depesz.com](http://explain.depesz.com/) 
[here](http://explain.depesz.com/s/70dd) so you can see the costs.
Building that CTE for every page load is expensive.

One way to get around this problem would be to materialize or denormalize the
data. This adds some storage overhead but should allow for much faster query
times.

Something like the following should suffice to store the data. The assumption
is that we will populate and maintain this relation with the status values that
we want which only leaves the `category_id` to filter by. Ideally you wouldn't
have to filter by the `category_id` either which would keep you from having
to compute the position each time.

{% highlight sql %}
CREATE TABLE object_container_order_rollup (
  container_id INTEGER REFERENCES containers(id),
  category_id INTEGER REFERENCES categories(id),
  object_id INTEGER REFERENCES objects(id),
  type VARCHAR(32) NOT NULL,
  index INTEGER NOT NULL,
  PRIMARY KEY (container_id, type, category_id, object_id)
);
CREATE INDEX object_container_order_rollup_idx 
ON object_container_order_rollup(index);
{% endhighlight %}

Then to populate the relation we can use the following.

{% highlight sql %}
WITH object_positions AS (
  SELECT object_container_map.container_id, objects.id, containers.category_id,
    row_number() OVER (PARTITION BY object_container_map.container_id
                      ORDER BY objects.id DESC) AS index
  FROM objects
  JOIN object_container_map ON (
    objects.id = object_container_map.object_id
    AND object_container_map.is_default = false
  )
  JOIN object_container_map default_map ON (
    objects.id = default_map.object_id
    AND default_map.is_default = true
  )
  JOIN containers ON (
    containers.id = default_map.container_id
  )
  WHERE objects.status = 1 AND containers.status = 1
   AND object_container_map.container_id > 500000
  ORDER BY objects.id DESC
)
INSERT INTO object_container_order_rollup
  (container_id, category_id, object_id, type, index)
SELECT container_id, category_id, id, 'id_desc', index
FROM object_positions;
{% endhighlight %}

Then we can use this query to get the tuples matching the categories
we want.

{% highlight sql %}
WITH container_500001 AS (
  SELECT container_id, category_id, object_id,
    row_number() OVER () AS index
  FROM object_container_order_rollup
  WHERE container_id = 500001 AND type = 'id_desc' AND
    category_id IN (2,3,4,5,6)
)
SELECT index, container_id, object_id, category_id
FROM container_500001
WHERE index BETWEEN 50000 - 6 AND 50000 + 6 
ORDER BY index ASC;
{% endhighlight %}

This works well and gives us a 20x speedup on a laptop (about 10x on production
hardware). The cost is about 80-100MB of data and indexes for the table and
the maintenance of the table which I won't go into here (changing categories,
removing those that don't match the status we are looking for, etc).

Another approach would be to use arrays and the PostgreSQL type system.
PostgreSQL allows you to create compound types which can be used as the
member type for arrays.

Here is an example. First we need to create the type. This will be the element
type for the array.

{% highlight sql %}
CREATE TYPE object_container_order_type AS (
  object_id INTEGER,
  category_id INTEGER
);
{% endhighlight %}

Next, we create the table to hold the array of objects in the heterogeneous
(by `category_id`) container. Each tuple will be uniquely identified by the
`container_id` and the `type`. The `type` will identify the type of ordering.

{% highlight sql %}
CREATE TABLE object_container_order (
  container_id INTEGER REFERENCES containers (id),
  type VARCHAR(32),
  order_objects object_container_order_type[],
  PRIMARY KEY (container_id, type)
);
{% endhighlight %}

Now we can populate the array with the tuples that match the status criteria so
that at query time we only need to to filter by the category. We'll also have
to maintain the arrays via triggers or a cron job to add or remove objects when
the status changes. One of the nice properties of using arrays like this is that
the updates to an array can be done atomically.

{% highlight sql %}
WITH object_positions AS (
  SELECT object_container_map.container_id, objects.id, containers.category_id,
   row_number() OVER (ORDER BY objects.id DESC) AS index
  FROM objects
  JOIN object_container_map ON (
    objects.id = object_container_map.object_id
    AND object_container_map.is_default = false
  )
  JOIN object_container_map default_map ON (
    objects.id = default_map.object_id
    AND default_map.is_default = true
  )
  JOIN containers ON (
    containers.id = default_map.container_id
  )
  WHERE objects.status = 1 AND containers.status = 1
)
INSERT INTO object_container_order (container_id, type, order_objects)
SELECT
  top.container_id,
 'id_desc',
  ARRAY(
        SELECT (id, category_id)::object_container_order_type
        FROM object_positions
        WHERE object_positions.container_id = top.container_id
        ORDER BY index ASC
       )
FROM object_positions AS top
GROUP BY top.container_id;
{% endhighlight %}

With the arrays populated by `container_id` we can now get the tuples
we want using `UNNEST`.

{% highlight sql %}
WITH container_500001 AS (
  SELECT (UNNEST(order_objects)).*
  FROM object_container_order
  WHERE container_id = 500001 AND type = 'id_desc'
), with_ordinal AS (
  SELECT object_id, category_id, row_number() OVER () AS index
  FROM container_500001
  WHERE category_id IN (2,3,4,5,6)
)
SELECT index, 500001, object_id, category_id
FROM with_ordinal
WHERE category_id IN (2,3,4,5,6)
    AND index BETWEEN 50000 - 6 AND 50000 + 6
ORDER BY index ASC;
{% endhighlight %}

This gives us about the same speedup as `object_container_order_rollup`
above but at one tenth the storage costs (8K vs 80MB). You can probably shave
off another 80% of the query cost by indexing directly into the array if you
don't have to do any filtering at display time.

To conclude, if you need to seek to a given position within a sorted relation
to fetch an object and it's neighbors you have a few options for materializing
the data. You can materialize the data in a table or an array and filter at
query time. Materializing to a table is straightforward should probably be the
default for most scenarios. Using a custom type and an array is also an option
but should be used [with caution and may not scale well with a lorge number of
elements](http://www.postgresql.org/docs/9.2/static/arrays.html#ARRAYS-SEARCHING).



In PostgreSQL 9.3 you could also create a materialized view and see if that
will give you the performance you need.

The SQL to setup the tables and the data can be found
[here](https://gist.github.com/drsnyder/9277054)
if you want to experiment with it. Here is the [SQL to create and populate the
rollup table](https://gist.github.com/drsnyder/9283214) and the [SQL to
create and populate the arrays](https://gist.github.com/drsnyder/9277366).

