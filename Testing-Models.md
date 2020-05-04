# Unit tests with Pytest

Django has a built in test runner and it works fine. But it is very verbose. I like [pytest](https://docs.pytest.org/en/latest/) in general for testing and we're going to use that to test this app.

## [Installing Pytest](https://github.com/dchess/mothra/commit/e8c88aad55397cbf0eb7695670631e23ab3b87be)

First, we need to add a new dependency to our code.

```bash
$ pipenv install pytest-django --skip-lock
```

Because our app dependencies have changed, we'll need to rebuild our docker image.

```bash
$ docker-compose build
```

We'll also want to add a config file for pytest so it knows where to find our Django settings and which files to run tests from. Create a file called `pytest.ini` in the root of the repo containing the following:

```ini
[pytest]
DJANGO_SETTINGS_MODULE = config.settings
python_files = tests.py test_*.py *_tests.py
```

## [Testing the Grade Model](https://github.com/dchess/mothra/commit/37b4d565f6e30a606661cb28c0d8dbc5dbf23c93)

Before we start writing our tests, it's worth thinking through what is worth testing. For models, I focus on testing constraints and validations. This allows me to confirm that the things I want to prevent in terms of user input are correctly handled. For our `Grade` model, we have two fields: `level` and `name`. Level must be between -1 and 12, while name should be no longer than 2 characters. Lastly, both are required fields that should not be blank. 

In the *accounts/tests.py* we'll add a few tests. Starting with checking that the string representation of the model conforms to the definition we set.

```python
import pytest
from accounts.models import Grade

pytestmark = pytest.mark.django_db


def test_grade_string_representation():
    grade = Grade.objects.create(level=0, name="K")
    assert str(grade) == grade.name
```

This code imports our model, configures it so that tests can use the database and then defines a test that adds a new grade object and asserts that casting the object to a string is equal to the name of the object.

To run this test:

```bash
$ docker-compose exec web pytest
```

It should return a nice little interface showing that your tests passed.

```bash
============================= test session starts ==============================
platform linux -- Python 3.7.3, pytest-5.2.1, py-1.8.0, pluggy-0.13.0
Django settings: config.settings (from ini file)
rootdir: /code, inifile: pytest.ini
plugins: django-3.6.0
collected 1 item                                                               

accounts/tests.py .                                                      [100%]

============================== 1 passed in 1.18s ===============================
```

If you wanted to be very strict about following [TDD](https://en.wikipedia.org/wiki/Test-driven_development) principles you would want to define this test first and then define your code. That way you write a test that fails, then write the code to make it pass. When I'm being a really good developer, I do this. But this tutorial isn't about how to follow best practices. It's about showing how I'm coding this thing in the real world. Don't judge me too harshly! If you want to test this for yourself, comment out the `__str__` method and run the test again.

### Fixtures

I'm going to be running a bunch of different tests. Most of which won't need to create a new object in the database, but rather test what happens when an attribute/field is set to something out of bounds. This is where a fixture comes in handy. I can setup my tests to use the same shared object and then modify it for each test.

```python
@pytest.fixture
def grade():
    return Grade.objects.create(level=0, name="K")
```

Then this fixture can be referenced in tests by passing it in. For more info on using fixtures check out the [pytest docs](https://docs.pytest.org/en/latest/fixture.html). 

```python
def test_grade_string_representation(grade):
    assert str(grade) == grade.name
```

For the next tests, we'll confirm that setting the grade level outside the minimum and maximum values throws a validation error. 

```python
import pytest
from accounts.models import Grade
from django.core.exceptions import ValidationError

pytestmark = pytest.mark.django_db

@pytest.fixture
def grade():
    return Grade.objects.create(level=0, name="K")


def test_grade_string_representation(grade):
    assert str(grade) == grade.name


def test_grade_min_level(grade):
    with pytest.raises(ValidationError):
        grade.level = -2
        grade.full_clean()
```

Notice how this new test sets up a [context manager](https://docs.pytest.org/en/latest/reference.html#pytest-raises) that encompasses the code that sets the grade level outside the minimum boundary. Any code executed in this context that raises a Validation error will satisfy the assertion. The call to `.full_clean()` is used to ensure the model [checks for validation](https://docs.djangoproject.com/en/2.2/ref/models/instances/#validating-objects).

For the rest of the tests, we'll follow this pattern.

```python
def test_grade_max_level(grade):
    with pytest.raises(ValidationError):
        grade.level = 13
        grade.full_clean()

def test_grade_name_max_length(grade):
    with pytest.raises(ValidationError):
        grade.name = "K" * 3
        grade.full_clean()
```

Another good thing to confirm with our tests is that both fields are required:

```python
def test_grade_level_is_required():
    with pytest.raises(IntegrityError):
        grade = Grade.objects.create(name="E")
        grade.full_clean()


def test_grade_name_is_required():
    with pytest.raises(ValidationError):
        grade = Grade.objects.create(level=0)
        grade.full_clean()
```

Notice that these tests don't use the fixture because we want to create a new object and confirm that the errors are thrown during creation. You might also spot that testing the omission of the grade level requires testing for an `IntegrityError` instead of a `ValidationError` that is due to the different data type and the check for a non-null value rather than a blank error. This is a database level validation and can be pulled in with an additional import.

```python
from django.db.utils import IntegrityError
```

Lastly, it's a good idea to pass some expected inputs and ensure they are able to be added without raising any errors. 

```python
def test_grade_correct_level(grade):
    for level in range(1, 13):
        grade.level = level
        grade.full_clean()

def test_grade_name_correct_length(grade):
    grade.name = "TK"
    grade.full_clean()
```

This is plenty of coverage for now. We'll run these test and confirm all pass and then move on to adding more models.

```bash
$ docker-compose exec web pytest
```

Which will return these results:

```bash
============================= test session starts ==============================
platform linux -- Python 3.7.3, pytest-5.2.1, py-1.8.0, pluggy-0.13.0
Django settings: config.settings (from ini file)
rootdir: /code, inifile: pytest.ini
plugins: django-3.6.0
collected 8 items                                                              

accounts/tests.py ........                                               [100%]

============================== 8 passed in 1.34s ===============================
```

Looks good!


