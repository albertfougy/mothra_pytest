# Rinse and Repeat

In the previous chapter, we saw how to create a model, turn it into a database migration, and test our model validations. We'll use this pattern for the remaining models in the accounts app.

## [Location](https://github.com/dchess/mothra/commit/264f7179d167ee36ca46e11d176f1f26b4bf104f)

Locations will represent the states that an organization operates in. This will help orgs identify partners operating in similar locations since many states have different compliance regulations. 

```python
class Location(models.Model):
    name = models.CharField(max_length=25)

    def __str__(self):
        return self.name

    class Meta:
        ordering = ("name",)
``` 

This will be a multi-select option in the account profile when setting up an organization. This will allow orgs to select multiple-states (especially important for charter orgs operating across states). One additional field could also be handy, which is the state's postal code abbreviation. So we'll add that as well.

```python
class Location(models.Model):
    name = models.CharField(max_length=25)
    abbreviation = models.CharField(max_length=2)

    def __str__(self):
        return self.name

    class Meta:
        ordering = ("name",)
```

Now, let's spin up the docker containers and make our migrations.

```bash
$ docker-compose up -d
$ docker-compose exec web python manage.py makemigrations
```

We should get a new migration file `0002_location.py` which we can apply to the database by running:

```bash
$ docker-compose exec web python manage.py migrate
```

And you should see:

```bash
Operations to perform:
  Apply all migrations: accounts, admin, auth, contenttypes, sessions
Running migrations:
  Applying accounts.0002_location... OK
```

## [Location Tests](https://github.com/dchess/mothra/commit/05c6a5ac7c03e3e5245d2a6855abb4dc9ea09476)

We've made this model quite simple and a few basic tests will suffice.

```python
from accounts.models import Location

@pytest.fixture
def location():
    return Location.objects.create(abbreviation="CA", name="California")


def test_location_name_max_length(location):
    with pytest.raises(ValidationError):
        location.name = "x" * 26
        location.full_clean()


def test_location_abbreviation_max_length(location):
    with pytest.raises(ValidationError):
        location.abbreviation = "x" * 3
        location.full_clean()


def test_location_name_is_required():
    with pytest.raises(ValidationError):
        location = Location.objects.create(abbreviation="CA")
        location.full_clean()


def test_location_abbreviation_is_required():
    with pytest.raises(ValidationError):
        location = Location.objects.create(name="California")
        location.full_clean()
```

To confirm these work, run:

```bash
$ docker-compose exec web pytest
```

## [Org Type](https://github.com/dchess/mothra/commit/e6c1bed228623043f7a876e70f20050d4e1befa8)

The last look up value we need to define before creating our organization model is the type. This will capture whether an organization is a charter or public district or some other kind of external entity.

One thing to keep in mind here are [python keywords](https://docs.python.org/3/library/functions.html). As nice as it would be to be able to call `organization.type` the `type()` function in python would make this confusing and potentially cause errors. So, redundancy aside, we will define this as `OrgType` instead. 

```python
class OrgType(models.Model):
    name = models.CharField(max_length=50)

    def __str__(self):
        return self.name

    class Meta:
        ordering = ("name",)
```

Then create the migration:

```bash
$ docker-compose exec web python manage.py makemigrations
$ docker-compose exec web python manage.py migrate
```

```bash
Operations to perform:
  Apply all migrations: accounts, admin, auth, contenttypes, sessions
Running migrations:
  Applying accounts.0003_orgtype... OK
```

## [Org Type Tests](https://github.com/dchess/mothra/commit/ec19d975077011c7fa1993d979f51585784d4df1)

Add two more tests for org type:

```python
from accounts.models import OrgType


@pytest.fixture
def org_type():
    return OrgType.objects.create(name="charter management organization")


def test_org_type_max_length(org_type):
    with pytest.raises(ValidationError):
        org_type.name = "x" * 51
        org_type.full_clean()


def test_org_type_name_is_required():
    with pytest.raises(ValidationError):
        org_type = OrgType.objects.create(name="")
        org_type.full_clean()
```

And confirm they pass:

```bash
$ docker-compose exec web pytest
```

We should now see:

```bash
======================== test session starts =========================
platform linux -- Python 3.7.3, pytest-5.2.1, py-1.8.0, pluggy-0.13.0
Django settings: config.settings (from ini file)
rootdir: /code, inifile: pytest.ini
plugins: django-3.6.0
collected 14 items                                                   

accounts/tests.py ..............                               [100%]

========================= 14 passed in 1.40s =========================
```

Notice that this is running all our tests, not just the new ones. This allows us to keep adding more coverage and confirming that none of the new additions breaks any of the previous code. This isn't an issue yet, but it can become one as the codebase grows.

Also notice that the platform is linux. I am running these on my macbook, but because the tests are running inside the docker container we're able to confirm the tests pass in the environment the app will be deployed too. This should cut down on the "it worked on my machine" errors that can come up during deployments.

## [Organization ](https://github.com/dchess/mothra/commit/a5de2f8b6e3b49353c9d73caa95f0791281acff9)

Time to add the final related model before creating our user profiles. The organization model will hold the org's name and links to the other three models we've defined so far. In addition, I'm adding a size field to potentially capture either number of schools or number of students (might refactor this later).

```python
class Organization(models.Model):
    name = models.CharField(max_length=50)
    size = models.IntegerField(null=True, blank=True)
    grades = models.ManyToManyField(Grade)
    locations = models.ManyToManyField(Location)
    org_type = models.ForeignKey(OrgType, on_delete=models.PROTECT)

    def __str__(self):
        return self.name

    class Meta:
        ordering = ("name",)
```

Couple of things to call out here. For `size` I'm adding a validation that allows the field to be blank. In some cases, adding a size may not make sense and we wouldn't want that to be a roadblock during account creation. The other is the `PROTECT` argument for the foreign key relation between `Organization` and `OrgType`. The default behavior of deleting a foreign key is a cascade delete trigger. I.E. if you delete an org type then all associated organizations are deleted as well. This is not the behavior we want. By adding setting it to protect the objects, this will throw an error forcing the admin to change the values before deleting. We could also opt to make them `NULL` instead, but I think this will help ensure better data quality and prevent unnecessary deletions.

We'll also add some additional admin portal functionality:

```python
from django.contrib import admin


class OrganizationAdmin(admin.ModelAdmin):
    list_filter = ("grades", "locations", "org_type")
```

This will allow us to use the lookup fields to filter the org list from the admin portal easily.

Lastly, we'll add all our new models to the admin portal by registering them.

```python
from django.contrib import admin
from .models import Grade, Location, OrgType
from .models import Organization, OrganizationAdmin


admin.site.register(Grade)
admin.site.register(Location)
admin.site.register(OrgType)
admin.site.register(Organization, OrganizationAdmin)
```

At this point, we run our migrations and add a few more tests.

```bash
$ docker-compose exec web python manage.py makemigrations
$ docker-compose exec web python manage.py migrate
```

## [Organization Tests](https://github.com/dchess/mothra/commit/19dd4801f4a82595a8c27ae03d5f1efda7d45bc5)

Now that we have our organization model defined, we'll add come tests. Because our model has foreign key relations as well as many-to-many relationships, we'll need to create a slightly more complicated fixture to test with. We can leverage some of the existing fixtures though.

```python
@pytest.fixture
def organization(location, org_type):
    # create test grades for HS (9-12)
    for grade in range(9, 13):
        Grade.objects.create(level=grade, name=f"{grade}")

    grades = Grade.objects.filter(level__gte=9)

    organization = Organization.objects.create(
        name="Test Organization", size=12, org_type=org_type
    )
    organization.grades.set(grades)
    organization.save()
    return organization
```

The first thing this does is create a grade record for grades 9-12. Then returns a queryset with all of them by filtering where the grade level is greater than or equal to 9. We then create a new organization object and set the grades relation to the result of the previous queryset. Notice how we pass in location and org_type as fixtures. 

Now we can write a few tests.

```python
def test_organization_name_max_length(organization):
    with pytest.raises(ValidationError):
        organization.name = "x" * 51
        organization.full_clean()


def test_organization_size_can_be_null(organization):
    organization.size = None
    organization.full_clean()


def test_organization_name_is_required(org_type):
    with pytest.raises(ValidationError):
        organization = Organization.objects.create(size=10, org_type=org_type)
        organization.full_clean()


def test_org_type_protected_delete(organization, org_type):
    with pytest.raises(ProtectedError):
        org_type.delete()


def test_org_type_is_required():
    with pytest.raises(IntegrityError):
        organization = Organization.objects.create(size=10, name="Test Org 2")
        organization.full_clean()


def test_organization_size_is_not_required(org_type):
    organization = Organization.objects.create(name="Test Org 3", org_type=org_type)
    organization.full_clean()
```
Note that for these tests to work, we need a couple new imports.

```python
from accounts.models import Organization
from django.db.models.deletion import ProtectedError
```

A common pattern for test naming is to use long names which describe what will be tested. In the process of defining these new tests, I notice that I'd forgotten to add a string representation test for location and org type. Yet another reason to follow TDD principles! 

### [Quick fix](https://github.com/dchess/mothra/commit/f2b38149efa9232acdb574bc09deec789efc4bee)

Let's go back and add those missing tests.

```python
def test_location_string_representation(location):
    assert str(location) == location.name

def test_org_type_string_representation(org_type):
    assert str(org_type) == org_type.name
```

And with that, we're done and can move on to the account profile.

```bash
$ docker-compose exec web pytest
```

## [Profile Model](https://github.com/dchess/mothra/commit/d89242437572b290407ccb8552f5a8ee62beb69f)

Finally, we can create our profile model. In it we will capture one additional data point (for now) beyond the default user attributes. In this case, we are interested in capturing the Github username for users to make connecting and sharing repos easier. We will also use the profile model to connect users to organizations. 

```python
from django.contrib.auth.models import User


class Profile(models.Model):
    user = models.OneToOneField(User, on_delete=models.PROTECT)
    github_id = models.CharField(max_length=39)
    organization = models.ForeignKey(Organization, on_delete=models.PROTECT)

    def __str__(self):
        return self.user.username

    class Meta:
        ordering = ("user",)
```

There are three fields created here. We are establishing a one-to-one relationship between profiles and users. That is to say that every user will have one profile and only one profile. We also create a foreign key relationship to organization. This is basically a many-to-one relationship. In that, many profiles can be linked to a single organization, but a profile can only have one organization. 

For validations, we add a max length to the github id. This seems arbitrary but came from a search of what limits Github places on username length.  We also add some deletion constraints preventing the deletion of the related models unless the profile is removed. We will not want to delete users (at all) but rather disable them, so this will prevent accidentally deleting profiles if a user is deleted, and it will also prevent orphaning profiles if an organization is deleted. 

Some additional functionality will be helpful in the admin portal for working with profiles. 

```python
class ProfileAdmin(admin.ModelAdmin):
    list_display = ("user", "organization")
    list_filter = ("organization",)
    search_fields = ["user", "organization"]
```

This will make sure filtering and searching for user profiles will be simpler. To register this in the admin portal we add the following to `admin.py`.

```python
from .models import Profile, ProfileAdmin


admin.site.register(Profile, ProfileAdmin)
```

Then we run our migrations.

```bash
$ docker-compose exec web python manage.py makemigrations
$ docker-compose exec web python manage.py migrate
```

## [Profile Tests](https://github.com/dchess/mothra/commit/f9644df92d29106f795ee1a19735f218c2b03bbd)

Given how slim the profile model is, there won't be many tests needed. However, we will need an extra fixture for the user in order to adequately test it.

```python
@pytest.fixture
def user():
    return User.objects.create_user("test_user", password="Testpassword")


@pytest.fixture
def profile(user, organization):
    return Profile.objects.create(
        user=user, github_id="test_user", organization=organization
    )


def test_profile_string_representation(profile):
    assert str(profile) == profile.user.username


def test_profile_github_id_max_length(profile):
    with pytest.raises(ValidationError):
        profile.github_id = "x" * 40
        profile.full_clean()


def test_profile_organization_is_required(profile):
    with pytest.raises(ValidationError):
        profile.organization = None
        profile.full_clean()

def test_profile_user_is_required(profile):
    with pytest.raises(ValidationError):
        profile.user = None
        profile.full_clean()
```

Not much new here except the [method for creating a user object](https://docs.djangoproject.com/en/2.2/ref/contrib/auth/#django.contrib.auth.models.UserManager.create_user). This is the preferred method for creating user objects, because it handles the password complexities. You can also pass an `email` parameter, but in this case we aren't using it for any tests. 

You can now run your tests and confirm they pass.

```bash
$ docker-compose exec web pytest
```

At this point we're done with all of our account models and tests. We can now move on to building out the data model for products that will be the core data we collect from users.  
