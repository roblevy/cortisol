# Cortisol: stress your database

## What is it?

**cortisol** creates a training ground for your DBA skills. It allows you to spin
up a random but realistic database, and random realistic users who are writing
to and reading from the database at potentially very high speeds.
This creates a potentially large, actively and concurrently used database for
you to monitor and debug live.

This will help you to stress-test your slow-query debugging skills, your settings
tuning skills, your OOM recovery powers etc. etc.

Cortisol can be set to run forever, or for a fixed-time or fixed-size
"experiment" which can safely be left running without eating your entire
storage. By default, experiments only last for one minute.

## Components

### Random database schema

An extension to SQLAlchemy ORM which allows you to specify a Faker provider to
use in the creation of fake data.

```python
class User(Base):
    name: Mapped[str] = generated_column(String(30), provider=fake.name)
    age: Mapped[str] = generated_column(
        provider=Provider(fake.pyint, min_value=18, max_value=120)
    )

with cortisol.session(sqlalchemy_engine) as session:
  session.create(User, 1000)
```

### Random data user roles

The writer will create random objects from your random schema either on a
schedule or as fast as possible.

```python
# create a User every 100ms with jitter
cortisol.writer([User], every_ms=100)
# update an existing record 10% of the time, and delete a record 1% of the time.
# The model will be selected randomly since no list was provided.
cortisol.writer(update_pct=10, delete_pct=1)
```

The reader will execute one of a set of automatically generated queries of
varying complexity. Each reader will randomly order the queries and prefer
queries near the start of the list.

The list of queries is generated along with the schema, and is written to
Cortisol's shared storage so that readers from different Python processes can
still read the same set of queries. (Or do we want to create these users in an
async event loop? Why would you want different processes doing this? There
might be a reason to make the stress more realistic.)

The set of queries can be replaced using `cortisol.refresh_experiment`. For
repeatable experiments, you just need to share the random-number generator seed
and the experiment will run in the same way again.

### Bounding the experiment

You can set limits on the extent of the experiment, causing all the users to
stop their activities when a certain point is reached:

```python
# Stop the experiment when there are 1 billion users
experiment = cortisol.experiment
experiment.seed(42)
experiment.add_limit.by_row_count(User, 1e9)
# Stop the experiment after two hours
experiment.add_limit.by_time(hours=2)
experiement.run()
```

Note that in each case, it might take a few moments for all the users to stop
their activities. This is because a signal needs to reach them.
