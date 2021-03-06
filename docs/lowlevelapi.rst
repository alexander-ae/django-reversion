.. _low-level-api:

Low-level API
=============

You can use django-reversion's API to build powerful version-controlled views outside of the built-in admin site.

**Please note:** The django-reversion API underwent a number of changes for the 1.5 release. The old API is now deprecated, and was removed in django-reversion 1.7. Documentation for the old API can be found on the :ref:`deprecated low-level API <deprecated-low-level-api>` wiki page.


Registering models with django-reversion
----------------------------------------

If you're already using the :ref:`admin integration <admin-integration>` for a model, then there's no need to register it. However, if you want to register a model without using the admin integration, then you need to use the ``reversion.register()`` method.

::

    import reversion

    reversion.register(YourModel)

**Warning:** If you’re using django-reversion in an management command, and are using the automatic ``VersionAdmin`` registration method, then you’ll need to import the relevant ``admin.py`` file at the top of your management command file.

**Warning:** When Django starts up, some python scripts get loaded twice, which can cause 'already registered' errors to be thrown. If you place your calls to ``reversion.register()`` in the ``models.py`` file, immediately after the model definition, this problem will go away.


Creating revisions
------------------

A revision represents one or more changes made to your models, grouped together as a single unit. You create a revision by marking up a section of code to represent a revision. Whenever you call ``save()`` on a model within the scope of a revision, it will be added to that revision.

**Note:** If you call ``save()`` outside of the scope of a revision, a revision is NOT created. This means that you are in control of when to create revisions.

There are several ways to create revisions, as explained below. Although there is nothing stopping you from mixing and matching these approaches, it is recommended that you pick one of the methods and stick with it throughout your project.


RevisionMiddleware
^^^^^^^^^^^^^^^^^^

The simplest way to create revisions is to use ``reversion.middleware.RevisionMiddleware``. This will automatically wrap every request in a revision, ensuring that all changes to your models will be added to their version history.

To enable the revision middleware, simply add it to your ``MIDDLEWARE_CLASSES`` setting as follows::

    MIDDLEWARE_CLASSES = (
        'django.contrib.sessions.middleware.SessionMiddleware',
        'django.contrib.auth.middleware.AuthenticationMiddleware', 
        'django.middleware.transaction.TransactionMiddleware',
        'reversion.middleware.RevisionMiddleware',
        # Other middleware goes here...
    )

Please note that ``RevisionMiddleware`` should go after ``TransactionMiddleware``. It is highly recommended that you use ``TransactionMiddleware`` in conjunction with ``RevisionMiddleware`` to ensure data integrity.


reversion.create_revision() decorator
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

If you need more control over revision management, you can decorate any function with the ``reversion.create_revision()`` decorator. Any changes to your models that occur during this function will be grouped together into a revision.

::

    @reversion.create_revision()
    def you_view_func(request):
        your_model.save()


reversion.create_revision() context manager
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

For Python 2.5 and above, you can also use a context manager to mark up a block of code. Once the block terminates, any changes made to your models will be grouped together into a revision.

::

    with reversion.create_revision():
        your_model.save()


Version meta data
-----------------

It is possible to attach a comment and a user reference to an active revision using the following method::

    with reversion.create_revision():
        your_model.save()
        reversion.set_user(user)
        reversion.set_comment("Comment text...")
    
If you use ``RevisionMiddleware``, then the user will automatically be added to the revision from the incoming request.

Custom meta data
^^^^^^^^^^^^^^^^

You can attach custom meta data to a revision by creating a separate django model to hold the additional fields. For example::

    from reversion.models import Revision

    class VersionRating(models.Model):
        revision = models.OneToOneField(Revision)  # This is required
        rating = models.PositiveIntegerField()

You can then attach this meta class to a revision using the following method::

    reversion.add_meta(VersionRating, rating=5)


Reverting to previous revisions
-------------------------------

To revert a model to a previous version, use the following method::

    your_model = YourModel.objects.get(pk=1)

    # Build a list of all previous versions, latest versions first:
    version_list = reversion.get_for_object(your_model)

    # Build a list of all previous versions, latest versions first, duplicates removed:
    version_list = reversion.get_unique_for_object(your_model)

    # Find the most recent version for a given date:
    version = reversion.get_for_date(your_model, datetime.datetime(2008, 7, 10))

    # Access the model data stored within the version:
    version_data = version.field_dict

    # Revert all objects in this revision:
    version.revision.revert()

    # Revert all objects in this revision, deleting related objects that have been created since the revision:
    version.revision.revert(delete=True)

    # Just revert this object, leaving the rest of the revision unchanged:
    version.revert()


Recovering Deleted Objects
--------------------------

To recover a deleted object, use the following method::

    # Built a list of all deleted objects, latest deletions first.
    deleted_list = reversion.get_deleted(YourModel)

    # Access a specific deleted object.
    delete_version = deleted_list.get(id=5)

    # Recover all objects in this revision:
    deleted_version.revision.revert()

    # Just recover this object, leaving the rest of the revision unchanged:
    deleted_version.revert()


Transaction Management
----------------------

django-reversion does not manage database transactions for you, as this is something that needs to be configured separately for the entire application. However, it is important that any revisions you create are themselves wrapped in a database transaction.

The easiest (and recommended) way to do this is by using the ``TransactionMiddleware`` supplied by Django. As noted above, this should go before the ``RevisionMiddleware``, if used.

If you want finer-grained control, then you should use the ``transaction.create_on_success`` decorator to wrap any functions where you will be creating revisions.


Advanced model registration
---------------------------

Following foreign key relationships
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Normally, when you save a model it will only save the primary key of any ForeignKey or ManyToMany fields. If you also wish to include the data of the foreign key in your revisions, pass a list of relationship names to the ``reversion.register()`` method.

::

    reversion.register(YourModel, follow=["your_foreign_key_field"])

**Please note:** If you use the follow parameter, you must also ensure that the related model has been registered with django-reversion.

In addition to ForeignKey and ManyToMany relationships, you can also specify related names of one-to-many relationships in the follow clause. For example, given the following database models::

    class Person(models.Model):
        pass

    class Pet(models.Model):
        person = models.ForeignKey(Person)

    reversion.register(Person, follow=["pet_set"])
    reversion.register(Pet)

Now whenever you save a revision containing a ``Person``, all related ``Pet`` instances will be automatically saved to the same revision.

Multi-table inheritance
^^^^^^^^^^^^^^^^^^^^^^^

By default, django-reversion will not save data in any parent classes of a model that uses multi-table inheritance. If you wish to also add parent models to your revision, you must explicitly add them to the follow clause when you register the model.

For example::

    class Place(models.Model):
        pass

    class Restaurant(Place):
        pass

    reversion.register(Place)
    reversion.register(Restaurant, follow=["place_ptr"])


Saving a subset of fields
^^^^^^^^^^^^^^^^^^^^^^^^^

If you only want a subset of fields to be saved to a revision, you can specify a ``fields`` or ``exclude`` argument to the ``reversion.register()`` method.

::

    reversion.register(YourModel, fields=["pk", "foo", "bar"])
    reversion.register(YourModel, exclude=["foo"])

**Please note:** If you are not careful, then it is possible to specify a combination of fields that will make the model impossible to recover. As such, approach this option with caution.


Custom serialization format
^^^^^^^^^^^^^^^^^^^^^^^^^^^

By default, django-reversion will serialize model data using the ``'json'`` serialization format. You can override this on a per-model basis using the format argument to the register method.

::

    reversion.register(YourModel, format="yaml")

**Please note:** The named serializer must serialize model data to a utf-8 encoded character string. Please verify that your serializer is compatible before using it with django-reversion.


Really advanced registration
^^^^^^^^^^^^^^^^^^^^^^^^^^^^

It's possible to customize almost every aspect of model registration by registering your model with a subclass of ``reversion.VersionAdapter``. Behind the scenes, ``reversion.register()`` does this anyway, but you can explicitly provide your own VersionAdapter if you need to perform really advanced customization.

::

    class MyVersionAdapter(reversion.VersionAdapter):
        pass  # Please see the reversion source code for available methods to override.

    reversion.register(MyModel, adapter_cls=MyVersionAdapter)


Automatic Registration by the Admin Interface
---------------------------------------------

As mentioned at the start of this page, the admin interface will automatically register any models that use the ``VersionAdmin`` class. The admin interface will automatically follow any InlineAdmin relationships, as well as any parent links for models that use multi-table inheritance.

For example::

    # models.py

    class Place(models.Model):
        pass

    class Restaurant(Place):
        pass

    class Meal(models.Model):
        restaurant = models.ForeignKey(Restaurant)

    # admin.py

    class MealInlineAdmin(admin.StackedInline):
        model = Meal

    class RestaurantAdmin(VersionAdmin):
        inlines = MealInlineAdmin,

    admin.site.register(Restaurant, RestaurantAdmin)

Since ``Restaurant`` has been registered with a subclass of ``VersionAdmin``, the following registration calls will be made automatically::

    reversion.register(Place)
    reversion.register(Restaurant, follow=("place_ptr", "meal_set"))
    reversion.register(Meal)

It is only necessary to manually register these models if you wish to override the default registration parameters. In most cases, however, the defaults will suit just fine.
