---
layout: post
title: When null nullifies default
excerpt: What happens when clients insert `null` in a column that should take a default value
modified: 2015-02-15
tags: [postgres, sql, default, null, ecto, elixir]
comments: true
---

Given a table `foos` created via the below query:

{% highlight sql %}
CREATE TABLE foos (
  id serial PRIMARY KEY,
  name text,
  ref_id text DEFAULT uuid_generate_v4()::text
);
{% endhighlight %}

Then you write an insert query to confirm the default behaves correctly:

{% highlight sql %}
insert into foos(name) values('bar');
{% endhighlight %}

Running a select query to confirms the insert works as expected:

{% highlight sql %}
soro=# select * from foos;
 id | name |                ref_id
----+------+--------------------------------------
  1 | bar  | 10cbf8ac-fbd6-45d8-918e-e913b9dbe2ca
{% endhighlight %}

Excited you start writing the client code via your ORM of choice(`ecto` for example) and find the below:

{% highlight sql %}
foo = Foo.Repo.insert(%Foo{name: "foo"})
foo = Repo.one from f in Foo, where f.id == ^foo.id
foo.id #=> 2
foo.name #=> "foo"
foo.ref_id #=> nil
{% endhighlight %}

:o Woohoo why is `foo.ref_id` `nil`, well the ORM sends `null` for properties not supplied.  
&nbsp;  
In this post we would examine 2 possible approaches to solving this problem:

- Generate a uuid by hand and pass in the `ref_id` e.g:

{% highlight sql %}
foo = Foo.Repo.insert(%Foo{name: "doe", ref_id: Ecto.UUID.generate})
foo.id #=> 3
foo.name #=> "doe"
foo.ref_id #=> "8C98BF51-2B4F-4AC6-A254-378A6666BB9C"
{% endhighlight %}

- Alternatively we can write a database trigger that overrides null and sets it to a `uuid` as expected, e.g:

{% highlight sql %}
CREATE OR REPLACE FUNCTION update_ref_id() RETURNS trigger AS $$
DECLARE
BEGIN
  IF NEW.ref_id IS NULL THEN
    NEW.ref_id = uuid_generate_v4();
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_refs
  BEFORE INSERT ON foos
  FOR EACH ROW EXECUTE PROCEDURE update_ref_id();
{% endhighlight %}

`BEFORE INSERT` guarantees that this trigger runs before the record is committed. So running the client code again:

{% highlight sql %}
foo = Foo.Repo.insert(%Foo{name: "Bob"})
foo = Repo.one from f in Foo, where f.id == ^foo.id
foo.id #=> 4
foo.name #=> "Bob"
foo.ref_id #=> "10cbf8ac-fbd6-45d8-918e-e913b9dbe2ca"
{% endhighlight %}

Boom! now our client code behaves just as expected.  
&nbsp;  
The above solution works with `ref_id` being `NOT NULL`  
&nbsp;  
Thanks for reading!
&nbsp;  
&nbsp;  
PS The queries above are all postgres specific but the idea should work across databases that comply with the sql standard.
