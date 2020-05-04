# Creating our app model

## Database migrations
Now that we've got our environment configured and ready for development, it's time to [migrate our database](https://docs.djangoproject.com/en/2.2/topics/migrations/). Even a brand new Django app has a bunch of built in migrations. That's one of the main appeals of using Django for web development. It's a batteries included web framework. Meaning it handles things like the [ORM](https://en.wikipedia.org/wiki/Object-relational_mapping) (object relational model), [authentication](https://docs.djangoproject.com/en/2.2/topics/auth/), [user management](https://docs.djangoproject.com/en/2.2/topics/auth/default/), and [permissions](https://docs.djangoproject.com/en/2.2/topics/auth/default/#permissions-and-authorization). It also comes with a full admin backend with record change history. 

Spin the docker network back up and then we'll issue a command to our running container directly to migrate the database schema for those built in admin models.

```bash
$ docker-compose up -d
$ docker-compose exec web python manage.py migrate
```

If you want to see how this has changed the database, you can connect to it with your [favorite database IDE](https://www.jetbrains.com/datagrip/) or just use good old `psql` from the command line.

```bash
$ docker-compose exec db psql -U mothra_admin -d mothra
```

It should look like this:

```bash
psql (11.4 (Debian 11.4-1.pgdg90+1))
Type "help" for help.

mothra=> 
```

You can then issue [commands](https://www.postgresql.org/docs/11/app-psql.html) directly to it. Try running `\d` to list all the database tables the migration created. Following by `\q` to quit. You should see the following:

```bash
                          List of relations
 Schema |               Name                |   Type   |    Owner     
--------+-----------------------------------+----------+--------------
 public | auth_group                        | table    | mothra_admin
 public | auth_group_id_seq                 | sequence | mothra_admin
 public | auth_group_permissions            | table    | mothra_admin
 public | auth_group_permissions_id_seq     | sequence | mothra_admin
 public | auth_permission                   | table    | mothra_admin
 public | auth_permission_id_seq            | sequence | mothra_admin
 public | auth_user                         | table    | mothra_admin
 public | auth_user_groups                  | table    | mothra_admin
 public | auth_user_groups_id_seq           | sequence | mothra_admin
 public | auth_user_id_seq                  | sequence | mothra_admin
 public | auth_user_user_permissions        | table    | mothra_admin
 public | auth_user_user_permissions_id_seq | sequence | mothra_admin
 public | django_admin_log                  | table    | mothra_admin
 public | django_admin_log_id_seq           | sequence | mothra_admin
 public | django_content_type               | table    | mothra_admin
 public | django_content_type_id_seq        | sequence | mothra_admin
 public | django_migrations                 | table    | mothra_admin
 public | django_migrations_id_seq          | sequence | mothra_admin
 public | django_session                    | table    | mothra_admin
(19 rows)
```

## Creating a Superuser
You could query any of them by passing a SQL command like `SELECT * FROM auth_user;` but at this point there will be no data to examine. 

Let's fix that by creating a superuser to administer our app with. 

```bash
$ docker-compose exec web python manage.py createsuperuser
```

You will be prompted to enter a username, email, and password for your user. Once created you can now log in to the Django admin site at `http://localhost/admin`.

After logging in you should see:

[[images/django_admin.png]]

Click on the users link and you'll see just what the Django admin portal offers us out of the box. We have built in search, filters, and forms for adding data. Clicking into our admin's user object shows us that not only does Django manage our user attributes like username and email but also has options to manage permissions (including groups) as well as tracking account creation and last login time. You can also see that Django handles password encryption with salted hashes using [SHA256](https://en.wikipedia.org/wiki/SHA-2). I'm glad I didn't have to code all that!

## Extending the User Model

Unfortunately there are additional attributes about our users we want to track. There are [many ways](https://simpleisbetterthancomplex.com/tutorial/2016/07/22/how-to-extend-django-user-model.html) to do this but I prefer the one-to-one model approach and will build out a `Profile` for users that will contain all the additional information we want to track.

## [Creating the Accounts App](https://github.com/dchess/mothra/commit/9ebf183a836ea6b497134cdd340fa0c8321f6339)

In the first chapter of this tutorial, we created a Django project but we don't have any apps yet. Confusing right? Well, Django organizes itself as an extensible platform. You can have several apps within a project that share common elements like authentication, but that have distinct self-contained models, views, and templates. To learn more about the MTV version of the MVC architecture pattern check out the [appendix](https://github.com/dchess/mothra/wiki/Appendix:-MTV-Archiecture). 

I find it helpful to organize parts of my Django project around these apps to distinguish the scope of responsibility. We'll start by creating an accounts app to track data related to user accounts.

```bash
$ docker-compose exec web python manage.py startapp accounts
```

This will create the placeholder files for our app code. You should see this series of files in your repo:

```bash
accounts/
├── __init__.py
├── admin.py
├── apps.py
├── migrations
│   └── __init__.py
├── models.py
├── tests.py
└── views.py

1 directory, 7 files
```

## [Adding the App Config to the Installed Apps](https://github.com/dchess/mothra/commit/73c1a290788c7bafd3fab3df07bf64ed23917e79)

In order for the app to be recognized by the project, we need to add it to our installed apps in the `settings.py` file:

```python
INSTALLED_APPS = [
    "django.contrib.admin",
    "django.contrib.auth",
    "django.contrib.contenttypes",
    "django.contrib.sessions",
    "django.contrib.messages",
    "django.contrib.staticfiles",
    "accounts.apps.AccountsConfig",
]
```

## The Accounts Data Model

As part of the requirements gathering step for this build, we determined what additional user information we wanted to capture for this app. This is represented in the entity relationship diagram (ERD) below along with how its relates to the `auth_user` model that is standard in Django.

[[images/accounts_erd.png]]

To create the models for our accounts app, we'll start from the right side of this diagram and work our way back to the user. That way we define each foreign key relationship before trying to reference it in the new model.

## [The Grade Model](https://github.com/dchess/mothra/commit/06879fdcecf29194889e037bd081f9feeb2ebf0b)

Our `models.py` will define the class blueprints for our data models. The [Django docs](https://docs.djangoproject.com/en/2.2/topics/db/models/) explain them as:

>A model is the single, definitive source of information about your data. It contains the essential fields and behaviors of the data you’re storing. Generally, each model maps to a single database table.

These blueprints will also generate migrations that will sync changes you make to your models into your database.

First we define the `Grade` model in `models.py`:

```python
from django.db import models


class Grade(models.Model):
    name = models.CharField(max_length=2)

    def __str__(self):
        return self.name

    class Meta:
        ordering = ("name",)
```

This creates a model called `Grade`. Gives it two fields: id (all models have an id implicitly) and name. And defines the data type of the name field as a string character with a max length of two. It also defines how the model will be displayed when called and how it will be sorted in the admin portal. 

A good question at this point might be, why store grade levels as strings? Shouldn't they be an integer? Well, yes but we'll get to that. For now, consider possibilities like kindergarten and transitional kindergarten which might complicate that. Sure we could capture those as 0 and -1 respectively, but will that be understood by users when filling out their organization's profile? Wouldn't K and TK be more clear? 

Use case will help define how we might need to capture these. Are we going to use these for grade ranges? I.E. grades between 1 and 5 for analysis purposes? We want to capture what grades an organization serves and how that impacts what products they choose. Thus enabling users to identify other organizations similar to themselves when looking for comparison. 

One way to handle that, is to provide multiple ways of thinking about the grade name. What if we provide a grade_level field as an int and a display_name field as string and get the best of both worlds? Let's do that before migrating.

```python
from django.db import models
from django.core.validators import MaxValueValidator, MinValueValidator


class Grade(models.Model):
    level = models.IntegerField(validators=[MaxValueValidator(12), MinValueValidator(-1)])
    name = models.CharField(max_length=2)

    def __str__(self):
        return self.name

    class Meta:
        ordering = ("level",)
```

We'll also provide some validations to prevent users from entering grades like 100 or -5, which would have no meaning for our use case. 

We'll also need to add this to our `admin.py` for it to show up in the admin portal.

```python
from django.contrib import admin
from .models import Grade


admin.site.register(Grade)
```

If we navigate to `http://localhost/admin` we'll see our new model. But if we click on it, we have a problem:

```bash
ProgrammingError at /admin/accounts/grade/
relation "accounts_grade" does not exist
LINE 1: SELECT COUNT(*) AS "__count" FROM "accounts_grade"
                                          ^
```

We've forgotten an important step! We need to migrate the database. But before that, we need Django to generate a migration script for us. 

```bash
$ docker-compose exec web python manage.py makemigrations
```

Which will result in this terminal message:

```bash
Migrations for 'accounts':
  accounts/migrations/0001_initial.py
    - Create model Grade
```

Let's look at the content of that migration script.

```python
# Generated by Django 2.2.6 on 2019-10-24 03:44

import django.core.validators
from django.db import migrations, models


class Migration(migrations.Migration):

    initial = True

    dependencies = [
    ]

    operations = [
        migrations.CreateModel(
            name='Grade',
            fields=[
                ('id', models.AutoField(auto_created=True, primary_key=True, serialize=False, verbose_name='ID')),
                ('level', models.IntegerField(validators=[django.core.validators.MaxValueValidator(12), django.core.validators.MinValueValidator(-1)])),
                ('name', models.CharField(max_length=2)),
            ],
            options={
                'ordering': ('level',),
            },
        ),
    ]
```

This was automatically generated for us with the previous `makemigrations` command and gives django instructions to use with the ORM when migrating our new schema to the database (independent of the dialect of SQL, postgres in our case). 

To execute this migration, we run:

```bash
$ docker-compose exec web python manage.py migrate
```

And we'll see this message:

```bash
Operations to perform:
  Apply all migrations: accounts, admin, auth, contenttypes, sessions
Running migrations:
  Applying accounts.0001_initial... OK
```

At this point, we can go into our admin portal and start entering grades into our accounts app. But before that, let's add some tests for our model to make sure it's working as intended.