---
title: How to constrain a JSONB field in Postgres
date: 2023-12-01T11:00:00+02:00
draft: false
tags: ["postgres"]
---

In Postgres, JSONB fields are very useful for storing semi-structured data. But sometimes the flexibility it offers can be a little bit scary. Imagine you want to store the result of some sort of inspection, which can lead to 2 different outputs, "create" or "update". Along with other logs. You expect your JSON field values to look like this:

```json
{"action": "create",
"some_useful_log_info": ...
}
``````

or

```json
{"action": "update",
"reason": ...
}
```

Your code expects the key `action` to exist, and the corresponding value to be `create` or `update`. Anything else will lead to problems.

Of course, you can test your code and make sure nothing else is saved in the database. But we know unexpected things happen. Manual modification of the database, a script to backfill some information, some holes in the test coverage, etc.

A database-level constraint is more reassuring.

And luckily for us, it is very easy to add a constraint on the database, making sure our semi-structured JSON looks like how we expect it.

Let's start with a simple constraint and make sure the key `action` exists in the JSON field `result`.

```sql
postgres=# ALTER TABLE jsontable ADD CONSTRAINT action_must_exist CHECK (result ? 'action');
```

We can also make sure the field is either empty or contains the `action` key:

```sql
postgres=# ALTER TABLE jsontable ADD CONSTRAINT action_must_exist CHECK (result = '"{}"' or result ? 'action');
```

But we can also be even more specific and ensure the result field is either empty or contains a key `action`, and in that case, the corresponding value must be `create`or `update`:

```sql
ALTER TABLE jsontable ADD CONSTRAINT action_must_exist_and_be_correct CHECK (result = '"{}"' or (result->>'action'=ANY ('{create,update}'::text[])) is true)
```

Note that you cannot only write ```result->>'action'=ANY ('{create,update}'::text[])```, because if the key `action` doesn't exist, `result->>'action'` is null and thus ```result->>'action'=ANY ('{create,update}'::text[])``` is also null. That's why you need the full `(result->>'action'=ANY ('{create,update}'::text[])) is true`.

With that constraint, you know that no matter how the data is updated, any unexpected value won't be saved in your table.
 