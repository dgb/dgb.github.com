---
layout: default
---
# Introduction
_This is a post I wrote while working at Shyp, originally [published on
Medium](https://medium.com/shyp-engineering/a-unique-journey-in-search-of-keys-3bb250471104)._

At Shyp we’ve begun undertaking an extensive maintenance project on our
primary API’s database, changing its tables’ primary keys from a
prefixed uuid stored as text, to utilizing the database’s native uuid
type, and adding the prefix in the application layer. This blog post
will give some general background on primary keys, outline why we made
this change, the process of preparing our application for this change,
and how we’ve gone about migrating the existing data.

# Primary Keys and UUIDs

Relational databases store records as rows in tables. Each table has a
strict definition of the columns that each row entry will be made up of,
along with their type. For example, you might have a contacts table
defined as:

    first: text
    last: text
    birthday: date

Then, you might have a table that looks like:

    | first   | last     | birthday   |
    |---------|----------|------------|
    | Lindsey | Homer    | 1984-06-24 |
    | Melvin  | Anderson | 1987-01-23 |
    | Jessie  | Bell     | 1957-09-16 |

In this case, the table does not have a primary key — the only way to
query it is based on the information it contains. Sometimes this is
appropriate, but often this becomes difficult when other information
needs to be associated with a record. For example, we might want to have
a notes table where we could add a note for each contact, but there’s no
way to clearly associate a particular contact with a note, as it’s
possible that two contacts can share the same name and birthday. So we
need a way of uniquely identifying the record with what’s called a
“key”. Some database designs use another attribute about a record, such
as a person’s Social Security Number, in order to provide a way of
looking up the record; this is called a “natural key”. However, this can
be problematic when we can’t guarantee that the value of the natural key
is truly unique, and so we need to utilize something else in order to
identify records.

Since relying on intrinsic properties of a dataset is problematic for
establishing the identity of records in the set, databases are often
designed using synthetic primary keys. These keys are guaranteed to be
unique by the database system, and are used for the sole purpose of
looking up a record. The most common approach is to use a sequential
integer. In our contacts example, this might look something like:

    | id | first   | last     | birthday   |
    |----|---------|----------|------------|
    | 1  | Lindsey | Homer    | 1984-06-24 |
    | 2  | Melvin  | Anderson | 1987-01-23 |
    | 3  | Jessie  | Bell     | 1957-09-16 |

When the responsibility of generating the primary keys is entirely in
the domain of the database system (as is often the case), this works
well. In a distributed system, however, you might want to have an
identifier for an entity without relying on the database to generate
that for you. In this case, you can take a randomized value and format
it in a consistent way; one category of these identifiers are known as
UUIDs, which are 128-bit values, usually formatted as a hyphen-separated
hexadecimal string. For example, “5c3ad9ab-a0fe-474c-87a2–38fc818d2b03”
is a UUID. There are several variants of UUIDs, but they all have the
same format, and the variants have to do with how the values are
generated. V4 UUIDs are truly random, and are what Shyp uses for primary
keys.

In addition, we prefix a short identifier to the id at Shyp. Although
this is a less common practice, it’s not a quirk that’s unique to us
(Stripe does this too). For example, a pickup is prefixed with “pik_”,
and a shipment is prefixed with “shp_”. This is useful from an
application and API perspective: Our operations apps can scan a QR code
containing the id of a number of things (a shipment, a container, a
warehouse worker’s badge) and route the UI appropriately. It’s also
useful for debugging, as we can readily know the kind of data we’re
dealing with just from its identifier.

These ids have historically been stored as text in our PostgreSQL
database. While this works and has no impact on lookup speed, it’s an
inefficient use of space. This is especially important with primary
keys, whose indexes should fit in memory. And while disk space is cheap
these days, RAM is comparatively expensive, and as we scale we’d like to
keep these costs under control. There is also no real reason to have the
database store these values as text, since PostgreSQL supports UUIDs as
a first class data type, and at the database level, you always know what
you’re querying for. The different representations are quite different
size-wise. A text representation of a uuid (with hyphens) is 36 bytes
(or 288 bits) — a little more than double the actual byte size of a
UUID. An index for these values reflects this difference as well, as we
will later see.

If there’s such a big difference, why did these get stored as text in
the first place? Shyp’s API was first built on Sails, and Sails’s ORM,
Waterline does not have built in support for handling UUIDs — it simply
treats them as text. We’ve maintained our own fork of Waterline, which
we’ve stripped a lot of features from, as well as removed all other
parts of Sails from our API. In any case, the first step in this
endeavor was to prepare our API and Waterline to handle UUIDs.

# Planning
Changing the type in the database, is fairly easy in some sense — simply change the type:

    ALTER TABLE users CHANGE COLUMN id SET TYPE uuid;

This sort of schema change could cause problems in a production
environment, particularly when the table is so large that running the
query will take more than a second or two, but it’s a good start in
terms of getting the application code prepared for using the prefix-less
ids.

In integrating with the existing codebase, we need to make sure the
assumptions around prefixes are being challenged. For example, in my
initial exploration I added a migration for one of our tables, ran the
unit tests, and everything passed (great, ship it!). Since I knew that
the prefixes weren’t being added I had to explicitly define the
assumptions around our prefixing behavior.

First, I wrote an integration test that attempted to create a record,
asserted that the newly created record was serialized by the application
with a prefix, and finally, could be searched with that record. This
ensures that the boundary around having prefixes or not is flexible;
Ideally we’d go with one or the other, but we have a living codebase
with many teams, and it’s impossible to change everything overnight
while continuing to deliver new features.

In other words, not only did we want to handle UUIDs, but also we
thought it would be nice if we could have our models handle prefixed
UUIDs, and simply ignore the prefix when running queries.

In order to handle this, we [added some UUID type coercion
functionality](https://github.com/Shyp/waterline/pull/32)
to our fork of Waterline. This was pretty straightforward, as Waterline
already has built in type coercion functionality; that is, if you give a
model a string for an integer primary key, it will attempt to coerce it
to an actual integer.

This results with the following Waterline query

    User.findOne('usr_abc123') // ...

only using the UUID in issuing the query

    SELECT * FROM users WHERE id = 'abc123';

instead of using the prefix

    SELECT * FROM users WHERE id = 'usr_abc123';

(The latter query would throw an error in Postgres, because the value is not a valid uuid).

Next, we needed to add the prefix in the models. This is relatively
straightforward; All models in our system have “toJSON()” invoked when
being serialized, so we simply override the id serialization here,
essentially adding the prefix if it’s not present on the id.

As a result of all this, we’ve got our models covered on both ends,
compatible with our prefixed UUIDs internally, and our integration test
passes. One nice quality of this approach in the model layer is that our
code will be compatible with the schema as we migrate.

# Data Migration

Now that our codebase will work with the prefix-less UUIDs, we need to
migrate the existing data. This is a little tricky, because while it
affects an entire table, it needs to happen without interrupting
service. It also needs to be easily reversed in case something goes
wrong.

To do this, we can break the migration into 5 steps:

1.	Create a new column that will be the eventual new id column.
2.	Fill any new records with the prefix-less id via a trigger on the table.
3.	Backfill this column with the prefix-less data.
4.	Create an index (concurrently) on that column.
5.	Swap the new column in, and keep the old one (in a transaction).

Our migration was on the “trackingevents” table, which records a
shipment’s various updates from the carrier. This is what the migration
looks like in SQL:

    ALTER TABLE trackingevents ADD COLUMN newid uuid;
    CREATE FUNCTION shyp_copy_newid() RETURNS TRIGGER AS $$
      BEGIN
        IF NEW.id IS NOT NULL THEN
          NEW.newid := regexp_replace(NEW.id, '\w+_', '')::uuid;
        END IF;
        RETURN NEW;
      END
    $$ LANGUAGE plpgsql;
    CREATE TRIGGER shyp_copy_newid BEFORE INSERT OR UPDATE ON trackingevents
      FOR EACH ROW EXECUTE PROCEDURE shyp_copy_newid();
    UPDATE trackingevents SET id = id;
    CREATE UNIQUE INDEX CONCURRENTLY trackingevents_newid_idx ON trackingevents(newid);
    CREATE UNIQUE INDEX CONCURRENTLY trackingevents_oldid_idx ON trackingevents(id);
    BEGIN;
      DROP TRIGGER shyp_copy_newid ON trackingevents;
      DROP FUNCTION shyp_copy_newid();
      ALTER TABLE trackingevents DROP CONSTRAINT trackingevents_pkey;
      ALTER TABLE trackingevents RENAME COLUMN id TO oldid;
      ALTER TABLE trackingevents ALTER COLUMN oldid DROP DEFAULT;
      ALTER TABLE trackingevents ALTER COLUMN oldid DROP NOT NULL;
      ALTER TABLE trackingevents RENAME COLUMN newid TO id;
      ALTER TABLE trackingevents ADD PRIMARY KEY USING INDEX trackingevents_newid_idx;
      ALTER INDEX trackingevents_newid_idx RENAME TO trackingevents_pkey;
      ALTER TABLE trackingevents ALTER COLUMN id SET DEFAULT gen_random_uuid();
    COMMIT;

Then if everything goes well, drop the old id column:

    ALTER TABLE trackingevents DROP COLUMN oldid;

And if something goes wrong, the old column can be swapped back in:

    BEGIN;
      UPDATE trackingevents SET oldid = 'trk_' || id WHERE oldid IS NULL;
      ALTER TABLE trackingevents DROP COLUMN id;
      ALTER TABLE trackingevents RENAME COLUMN oldid TO id;
      ALTER TABLE trackingevents ALTER COLUMN id SET DEFAULT 'trk_' || gen_random_uuid();
      ALTER TABLE trackingevents ADD PRIMARY KEY USING INDEX trackingevents_oldid_idx;
      ALTER INDEX trackingevents_oldid_idx RENAME TO trackingevents_pkey;
    COMMIT;

Following those steps, we migrated the table. The resulting index size was a little less than half the existing one. Not bad!

# Next Steps
There’s been a lot of work going into this. We’ve migrated two of our
largest tables, and any new tables use uuids for their primary keys.

As of writing our database has 67 tables, ten of which have uuid primary
keys.

Some of these have foreign keys, so the foreign key needs to be changed
as well. The setup, such as backfilling to a temporary column, will be
the same, but the transaction of swapping the columns out would be
slightly different. Let’s pretend a table called “trackingeventdetails”
existed and had a foreign key pointed at the “trackingevents” table’s
id. We’d have to write something like:

    BEGIN;
    -- New! Drop the foreign key reference to the table
    ALTER TABLE trackingeventsdetails DROP CONSTRAINT "trackingeventdetails_trackingEventId_fkey";
    -- Same as above
    ALTER TABLE trackingevents DROP CONSTRAINT trackingevents_pkey;
    ALTER TABLE trackingevents RENAME COLUMN id TO oldid;
    ALTER TABLE trackingevents ALTER COLUMN oldid DROP DEFAULT;
    ALTER TABLE trackingevents ALTER COLUMN oldid DROP NOT NULL;
    ALTER TABLE trackingevents RENAME COLUMN newid TO id;
    ALTER TABLE trackingevents ADD PRIMARY KEY USING INDEX trackingevents_newid_idx;
    ALTER INDEX trackingevents_newid_idx RENAME TO trackingevents_pkey;
    ALTER TABLE trackingevents ALTER COLUMN id SET DEFAULT gen_random_uuid();
    -- Now we have to migrate the table pointing here as well
    -- (pretend we have a backfilled column)
    ALTER TABLE trackingeventdetails RENAME COLUMN "trackingEventId" TO oldTrackingEventId;
    ALTER TABLE trackingeventdetails RENAME COLUMN "newTrackingEventId" TO id;
    ALTER TABLE trackingeventdetails
      ADD CONSTRAINT "trackingeventdetails_trackingEventId_fkey"
      FOREIGN KEY REFERENCES trackingevents(id);
    COMMIT;

Overall, it’s pretty similar — we just add the modifications to the
foreign keys, and since it’s in a transaction, to the outside it appears
as if nothing happened.

There are more complicated situations we can get into with regard to our
foreign keys, but thankfully these are on our smaller tables. If you
have any questions or improvements, please let us know! Also, a shout
out to Braintree for posting their [summary of high volume operations in
Postgres](https://www.braintreepayments.com/blog/safe-operations-for-high-volume-postgresql/).
It’s been a great resource for this project as well as some of our
ongoing feature work.
