Combine LDAP and classical authentication in django 
####################################################

:date: 2013-12-20
:tags: django, ldap
:category: python
:author: Florent Lebreton (fle)

Still in our CAMM project `JOB <http://makina-corpus.com/realisations/application-de-gmao>`_ at `Makina Corpus <http://makina-corpus.com>`_, we needed to provide an authentication system through a LDAP server. The module `django-auth-ldap <http://https://pypi.python.org/pypi/django-auth-ldap>`_ does most of work itself but we decided to combine this LDAP authentication with the classical django authentication for two reasons:

* Keep possibility to create users only in django without having them in the LDAP directory (mainly for `superusers`);
* Be able to fallback on django authentication for all users in case of a LDAP server failure.

So to handle this, we have to:

* Differentiate LDAP directory users from django database users;
* Store password of LDAP users in django to be able to switch on classical authentication;
* Implement two custom authentication backends.

User model
----------

We just implement a custom model user with a boolean ``from_ldap`` to diffentiate LDAP users from classical django users.

.. code-block:: python

    class MyUser(AbstractUser):
        from_ldap = models.BooleanField(
            _('LDAP user'),
            editable=False,
            default=False)

LDAP authentication backend
---------------------------

This LDAP backend has two goals :

* Save user password in django database, thus he'll be able to log in django if LDAP authentication backend is disabled;
* Force ``from_ldap`` field to ``True`` when a user is created by this way.

.. code-block:: python

    from django_auth_ldap.backend import LDAPBackend
    from django.contrib.auth import get_user_model

    class MyLDAPBackend(LDAPBackend):
        """ A custom LDAP authentication backend """

        def authenticate(self, username, password):
            """ Overrides LDAPBackend.authenticate to save user password in django """

            user = LDAPBackend.authenticate(self, username, password)

            # If user has successfully logged, save his password in django database
            if user:
                user.set_password(password)
                user.save()

            return user

        def get_or_create_user(self, username, ldap_user):
            """ Overrides LDAPBackend.get_or_create_user to force from_ldap to True """
            kwargs = {
                'username': username,
                'defaults': {'from_ldap': True}
            }
            user_model = get_user_model()
            return user_model.objects.get_or_create(**kwargs)


Classical authentication backend
--------------------------------

We override ``django.contrib.auth.backends.ModelBackend`` to ensure LDAP users can't logged in with this one if ``MyLDAPBackend`` is available.

.. code-block:: python

    from django.contrib.auth import get_backends, get_user_model
    from django.contrib.auth.backends import ModelBackend

    class MyAuthBackend(ModelBackend):
        """ A custom authentication backend overriding django ModelBackend """

        @staticmethod
        def _is_ldap_backend_activated():
            """ Returns True if MyLDAPBackend is activated """
            return MyLDAPBackend in [b.__class__ for b in get_backends()]

        def authenticate(self, username, password):
            """ Overrides ModelBackend to refuse LDAP users if MyLDAPBackend is activated """

            if self._is_ldap_backend_activated():
                user_model = get_user_model()
                try:
                    user_model.objects.get(username=username, from_ldap=False)
                except:
                    return None

            user = ModelBackend.authenticate(self, username, password)

            return user


Django settings and fallback solution
--------------------------------------

Normally, we have our two backends activated :

* LDAP users can only connect through ``MyLDAPBackend``
* Django users can connect through ``MyAuthBackend``

.. code-block:: python

    AUTHENTICATION_BACKENDS = (
        'accounts.backends.MyLDAPBackend',
        'accounts.backends.MyAuthBackend',
    )

In case of a LDAP directory failure, we just have to disable ``MyLDAPBackend`` and everybody can connect with ``MyAuthBackend``.

.. code-block:: python

    AUTHENTICATION_BACKENDS = (
        #'accounts.backends.MyLDAPBackend',
        'accounts.backends.MyAuthBackend',
    )

Thanks `leplatrem <http://twitter.com/leplatrem>`_ for review!