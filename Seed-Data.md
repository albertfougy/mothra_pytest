# Seeding the Database

Before moving on to building out the data model for our product inventory, let's load some default data for the organization lookups (grade, location, org type). That way our app will always come loaded with some initial values for the these models.

Spin up docker-compose again:

```bash
$ docker-compose up -d
```

## Fixtures

One of the simplest ways to load data into a Django app is through the [django-admin loaddata command](https://docs.djangoproject.com/en/2.2/ref/django-admin/#loaddata). It enables populating a model with data from a `fixture`.

Django defines a fixture as:

>a collection of files that contain the serialized contents of the database. Each fixture has a unique name, and the files that comprise the fixture can be distributed over multiple directories, in multiple applications.

We'll create a serialized file (json) for each model. 

### [Fixtures](https://github.com/dchess/mothra/commit/fb9c190e36bf73dfb42aa0b24d284ba1e16d6a8b)

We'll create a new folder in the accounts directory called **fixtures** and add a file called `grades.json`.

```json
[
    {
        "model": "accounts.grade",
        "pk": 1,
        "fields": {
            "level": -1,
            "name": "TK"
        }
    },
    {
        "model": "accounts.grade",
        "pk": 2,
        "fields": {
            "level": 0,
            "name": "K"
        }
    },
    {
        "model": "accounts.grade",
        "pk": 3,
        "fields": {
            "level": 1,
            "name": "1"
        }
    }
]
```

Notice the format of these files. There is a list of dictionaries, each containing a reference to the model, a primary key, and the values of each field in the model to be populated. The code excerpt above cuts off after first grade, but see the commit for the full file definition.

To load this, we could simply run:

```bash
$ docker-compose exec web python manage.py loaddata --app accounts grades.json
```

But we would need to remember to run this during setup of the application. Instead, we'll skip this step and add some custom migrations to make sure this happens automatically during database migration. However, first we'll continue this for the other fixtures by creating two more files `locations.json` and `org_types.json` in the same new fixture directory. 

💡**Optional**: If writing large json files in the specific fixture format is too burdensome, you can check out [this utility script](https://github.com/kipp-bayarea/django_db_seed) I wrote to convert simple csv files into the necessary format for Django fixtures. 

## [Custom Migrations](https://github.com/dchess/mothra/commit/1f0267e81512f5d8c4dabad809a93c93c1595d69)

We can leverage [the migrations functionality within Django](https://docs.djangoproject.com/en/2.2/topics/migrations/#data-migrations) to seed our database with the fixture data automatically. Normally to generate a migration after defining a model we'd run `python manage.py makemigrations` but we can override this functionality to define our own custom migrations.

```bash
$ docker-compose exec web python manage.py makemigrations --empty accounts --name seed_grade
```

This will create a new empty migration file called `0006_seed_grade.py` in the migrations folder.

It will look like this:

```python
from django.db import migrations


class Migration(migrations.Migration):

    dependencies = [
        ("accounts", "0005_auto_20191028_0032")
    ]

    operations = [
    ]
```

We can then extend this using the [RunPython](https://docs.djangoproject.com/en/2.2/ref/migration-operations/#runpython) function.

```python
import os
from django.db import migrations
from django.core.management import call_command

fixture_dir = os.path.abspath(os.path.join(os.path.dirname(__file__), "../fixtures"))
fixture_filename = "grades.json"


def load_fixture(apps, schema_editor):
    fixture_file = os.path.join(fixture_dir, fixture_filename)
    call_command("loaddata", fixture_file)


def unload_fixture(apps, schema_editor):
    "Brutally deleting all entries for this model..."

    MyModel = apps.get_model("accounts", "Grade")
    MyModel.objects.all().delete()


class Migration(migrations.Migration):

    dependencies = [
        ("accounts", "0005_auto_20191028_0032")
    ]

    operations = [
        migrations.RunPython(load_fixture, reverse_code=unload_fixture)
    ]
```

A couple of things are going on here and the code is pulled from [this StackOverflow question](https://stackoverflow.com/questions/25960850/loading-initial-data-with-django-1-7-and-data-migrations). It defines which fixture file to use. Defines two functions for calling the loaddate functionality on that fixture and then a rollback function (to reverse the migration, if need be) that simply deletes all data in that model. **Note** this is flagged as *brutal* because it will not limit the deletion to just the data created with the fixture, but any data in that model at the time the migration is reversed. So if additional records had been added manually, they would also be dropped. 

We can then repeat this migration for the other two fixtures simply changing the fixture and model references in the code. There is likely a way to reduce the code duplication by either merging the fixtures into a single seed file or perhaps abstracting the migrations, but I like how atomic this makes the migrations and allows for more modularity in how they are applied. 

You could also forgo this step entirely, and simply load the fixture manually with the `loaddata` command as needed. 

Now we're ready to build out the product inventory.

```bash
$ docker-compose down
```

## Merging a Pull Request

This whole time I've been writing this code on a feature branch called `accounts`. Before moving on to products, I'm going to open a Pull Request in Github and merge this code into the `master` branch. 

