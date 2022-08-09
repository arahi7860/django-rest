[![General Assembly Logo](https://camo.githubusercontent.com/1a91b05b8f4d44b5bbfb83abac2b0996d8e26c92/687474703a2f2f692e696d6775722e636f6d2f6b6538555354712e706e67)](https://generalassemb.ly/education/web-development-immersive)

# Django Rest API

As you've seen so far Django is a framework that's set up to do a lot of the
lower level work for you when creating a project to allow you to focus more
abstract, higher level work. The [Django REST
Framework](https://www.django-rest-framework.org/) applies that same philosophy
to the creation of RESTful APIs. 

Let's jump in and set up a simple API to see how it works!

## Setting Up Our Django Project
We're going to be making a new Django project called `ga` that will keep track
of General Assembly Students and Cohorts. Lets run `mkdir ga` in our projcets
folder to give our project a home. 

Next, we can set up the projects virtual environment with `virtualenvwrapper`
with the command `mkvirtualenv ga`. Once that's done, we have our always
necessary step: `pip freeze > requirements.txt` to have that record of our
dependencies.

Now, lets use `django-admin startproject ga .` to have django create our
project. **Don't forget the dot!** It will automatically add a `db.sqlite3` file
at the root level that we don't need so we can `rm` that sucker.

## Creating Our Database
When we've created databases in the past, we've opened up `psql` and entered our
commands there. Let's do something a little different this time, which is to
create a file that has our commands, then execute that file.

Create a file called `create-database.sql` and fill it with the following:

```sql
CREATE DATABASE ga;

CREATE USER ga_admin WITH PASSWORD 'password';

GRANT ALL PRIVILEGES ON DATABASE ga TO ga_admin;

```
We can execute that script by running `psql -f create-database.sql` from the
command line. If you check the `man` page for `psql`, you'll see that the `-f`
option (or `--file`) is used when you want to read commands from whatever file
is passed as the next argument. 

> The end result is identical to what would happen if you entered those commands
inside the `psql` shell. The advantage of writing the script into a file is, of
course, we can use it again! Normally on a project like this, we would delete
the `create-database.sql` at this point but you should keep (and maybe add a
comment on how to execute the file) for reference sake.

Next, let's install the Postgresql driver with `pip install psycopg2-binary` and
follow it up with a `pip freeze > requirements.txt`.

This is a good time to check and see if we have any errors so lets see if there
are any problems when we run the server. Normally we would type out `python
manage.py runserver` to spin it up but that's _so many letters_ and we are _so
lazy_. When working with Django, you'll find yourself typing `python manage.py`
very often, which is a good indicator that you could benefit from an alias.

If you like, you could throw the following alias into your zshrc/bashrc to save
your hands some stress:

```bash
alias pm='python manage.py`
```

Don't forget to run `source ~/.zshrc` afterward!

Now that our server is running smoothly, we can run our migrations to make sure
our database is set up properly. Because this is our initial migration, we don't
need to run `makemigrations` first. We can skip ahead to `python manage.py
migrate`. Later on, when we've made any new models or made changes to existing
ones, we'll have to run `python manage.py makemigrations` followed by `python
manage.py migrate`.

## Creating Our App
We're going to add an application called 'students' to our project. The first
step is to use `django-admin startapp students` to create all of the necessary
files.

Then, we need to open the `settings.py` file in our project folder and add
`'students'` to the list of `INSTALLED_APPS`. 

With that done, we can define our models in `students/models.py`. We're going to
make a `Cohort` model with a name and subject field and we're going to make a
`Student` model with a name field that's going to be related to a `Cohort`.

```py
class Cohort(models.Model):
    name = models.CharField(max_length=128)
    subject = models.CharField(max_length=5)

class Student(models.Model):
    name = models.CharField(max_length=128)
    cohort = models.ForeignKey(
        Cohort,
        on_delete=models.CASCADE,
        related_name='students'
    )
```

A couple of notes:
- The `on_delete` argument is there to define specific behavior that should
take place when an instance of a `Cohort` is deleted. We've provided the
instruction `models.CASCADE`. That means that when the referenced object is
deleted, it will also delete the objects that have a reference to it. 

Why? Imagine you had a blog post with a number of comments attached to it. If
the blog post was deleted, you wouldn't want to keep the comments floating
around attached to nothing, would you?

- The `related_name` is what name the `Student` model is given from the
perspective of the `Cohort`.

## Value Validation

We can limit the acceptable values for `Cohort` subjects by passing in a
`choices` argument to any of our fields. Choices is going to be a list of tuples
where the first value is how we'd like to store our choice in the database and
the second is what we'd like to be displayed in the admin panel. 

Add the following above the model definitions in `students/models.py`:

```py
COHORT_SUBJECTS = [
    ('SEI', 'Software Engineering Immersive'),
    ('UXDI', 'User Experience Development Immersive'),
    ('DSI', 'Data Science Immersive'),
]
```

Then change the `Cohort` model so that it looks like this:

```py
class Cohort(models.Model):
    name = models.CharField(max_length=128)
    subject = models.CharField(max_length=5, choices=COHORT_SUBJECTS)
```

Because we've made a change to our model, we're going to need to make and run a
migration with `python manage.py makemigrations` and `python manage.py migrate`.

Next, lets register our models by jumping into `students/admin.py` and editing
it to look like this:

```py
from django.contrib import admin
from .models import Cohort, Student

# Register your models here.
admin.site.register(Cohort)
admin.site.register(Student)
```
Finally, we need to create our superuser. Run `python manage.py createsuperuser`
and run through the prompts.

Now let's load up the page and create some cohorts!

You may notice that once a cohort is created, it's listed in the admin panel as
"Cohort Object" rather than saying anything useful. That's because _waaaaay_
back in the day when we created our models, we didn't add any `__str__` methdod
to describe what should be returned when we try to print our models.

If jump back into the `students/models.py` file and add a nice definition...

```py
def __str__(self):
    return '{self.subject} - {self.name}'
```
... then refresh the page, we get a nice formatted string. Note that we can do
any sort of valid python for the `__str__` method, meaning we can make our
returns as simple or as fancy as we like. Go ahead and make a `__str__` for the
`Student` model too!

## Serializer
Now we can set up a serializer! From the Django REST documentation:

>Serializers allow complex data such as querysets and model instances to be converted to native Python datatypes that can then be easily rendered into JSON, XML or other content types. Serializers also provide deserialization, allowing parsed data to be converted back into complex types, after first validating the incoming data.

- set up serializer by creating serializer.py in students folder

```py
from rest_framework import serializers
from .models import Cohort, Student

class CohortSerializer(serializers.ModelSerializer):
    class Meta:
        model = Cohort
        fields = '__all__'

class StudentSerializer(serializers.ModelSerializer):
    class Meta:
        model = Student
        fields = '__all__'
```

- set up views.py
    - views are basicallly funcitons that get called at our endpoitsn
    - the django.shortcuts import render bit that's in there at the start is
    unnecessary for us. It renders takes a model and renders it with an HTML
    template. Instead we will have this:
    
```py
from rest_framework import viewsets

from .serializers import CohortSerializer, StudentSerializer
from .models import Cohort, Student

# Create your views here.

class CohortViewSet(viewsets.ModelViewSet):
    queryset = Cohort.objects.all()
    serializer_class = CohortSerializer
    

class StudentViewSet(viewsets.ModelViewSet):
    queryset = Student.objects.all()
    serializer_class = StudentSerializer
```

- set up our URLs
    - you could make a URLs file inside of the students app and then import it
    into the urls file at the project level folder but we're just going to throw
    them into the ga/urls.py because our project only has a single app.

- in ga/urls.py:

```py
from django.contrib import admin
from django.urls import path, include

from rest_framework import routers
from students.views import CohortViewSet, StudentViewSet

router = routers.DefaultRouter()
router.register(r'cohort', CohortViewSet)
router.register(r'student', StudentViewSet)


urlpatterns = [
    path('', include(router.urls))
    path('admin/', admin.site.urls),
]
```

This router takes the place of all the endpoints I'd have to write by hand

That's it! We can test it all out in postman if we like

This is the start of the journey. You'll have to pick up the rest from the
[Django REST Framework](https://www.django-rest-framework.org/) documentation. Just as a taste of some of the things you
can do, lets change our `StudentSerializer` in our `serializers.py` to look like
this and see what happens:

```py
class StudentSerializer(serializers.HyperlinkedModelSerializer):
...
```
