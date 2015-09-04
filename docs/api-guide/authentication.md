source: authentication.py

# Authentication Аутентификация

> Auth needs to be pluggable. Аутентификация должна быть подключаемой
>
> &mdash; Jacob Kaplan-Moss, ["REST worst practices"][cite]

Authentication is the mechanism of associating an incoming request with a set of identifying credentials, such as the user the request came from, or the token that it was signed with.  The [permission] and [throttling] policies can then use those credentials to determine if the request should be permitted.
Аутентификация это механизм ассоциации входящего запроса с набором определенных полномочий, таких как откуда пришел пользователь и запрос, или как был подписан токен. Разрешения и ограничения прав могут использовать те полномочия чтобы определить есть ли разрешение у запроса.

REST framework provides a number of authentication schemes out of the box, and also allows you to implement custom schemes.
Рест фраймворк предоставляет несколько аутенфикационных схем из коробки, и также позволяет вам использовать свои схемы.

Authentication is always run at the very start of the view, before the permission and throttling checks occur, and before any other code is allowed to proceed.
Аутентификация всегда запускается в начале view, до того как происходить проверка разрешений и ограничений, и до того как начнет выполнятся другой код.

The `request.user` property will typically be set to an instance of the `contrib.auth` package's `User` class.
Свойство `request.user` обычно установлено в экземпляре `contrib.auth` пакета `User` 

The `request.auth` property is used for any additional authentication information, for example, it may be used to represent an authentication token that the request was signed with.
Свойство `request.auth` используется для любой дополнительной аитентификационной информации, для примера, оно может использовать для предоставления аутификационного токена которым будет подписан запрос.

---

**Note:** Don't forget that **authentication by itself won't allow or disallow an incoming request**, it simply identifies the credentials that the request was made with.
**Примечание** Не забывайте, что **аутентификация сама по себе не позволяет или запрещает входящие запросы**, она просто определяет полномочия которые даны запросу.

For information on how to setup the permission polices for your API please see the [permissions documentation][permission].
Для информации о том как установить политику разрешений для вашего API пожалуйста смотрите [документацию разрешений][permission].
---

## How authentication is determined | Как аутентификация определяется

The authentication schemes are always defined as a list of classes.  REST framework will attempt to authenticate with each class in the list, and will set `request.user` and `request.auth` using the return value of the first class that successfully authenticates.
Аутификационные схемы всегда определяются как список классов. REST fremework будет пытаться авторизоваться к каждому классу в списке, и будет устанавливать `request.user` и `request.auth` используя возвращаемое значение первого класса который успешно авторизуется.

If no class authenticates, `request.user` will be set to an instance of `django.contrib.auth.models.AnonymousUser`, and `request.auth` will be set to `None`.
Если не один класс не авторизуется, `request.user` будет установлен в экземпляр `django.contrib.auth.models.AnonymousUser`, и `request.auth` будет установлен как `None`.

The value of `request.user` and `request.auth` for unauthenticated requests can be modified using the `UNAUTHENTICATED_USER` and `UNAUTHENTICATED_TOKEN` settings.
Значение `request.user` и `request.auth` для неавторизованых запросов может быть изменено с помощью `UNAUTHENTICATED_USER` и `UNAUTHENTICATED_TOKEN` настроек.

## Setting the authentication scheme | Настройки авторизационной схемы

The default authentication schemes may be set globally, using the `DEFAULT_AUTHENTICATION_CLASSES` setting.  For example.
По умолчанию авторизационная схема может быть установленна глобально, в `DEFAULT_AUTHENTICATION_CLASSES` файла settings.py

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': (
            'rest_framework.authentication.BasicAuthentication',
            'rest_framework.authentication.SessionAuthentication',
        )
    }

You can also set the authentication scheme on a per-view or per-viewset basis,
using the `APIView` class based views.
Также вы можете установить авторизационную схему для каждого view отдельно для views наследованных от `APIView`

    from rest_framework.authentication import SessionAuthentication, BasicAuthentication
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        authentication_classes = (SessionAuthentication, BasicAuthentication)
        permission_classes = (IsAuthenticated,)

        def get(self, request, format=None):
            content = {
                'user': unicode(request.user),  # `django.contrib.auth.User` instance.
                'auth': unicode(request.auth),  # None
            }
            return Response(content)

Or, if you're using the `@api_view` decorator with function based views.
Или, если вы используете `@api_view` декоратор

    @api_view(['GET'])
    @authentication_classes((SessionAuthentication, BasicAuthentication))
    @permission_classes((IsAuthenticated,))
    def example_view(request, format=None):
        content = {
            'user': unicode(request.user),  # `django.contrib.auth.User` instance.
            'auth': unicode(request.auth),  # None
        }
        return Response(content)

## Unauthorized and Forbidden responses | Неавторизованные и Запрещенные ответы

When an unauthenticated request is denied permission there are two different error codes that may be appropriate.
Когда неавторизованному запросу отказано в разрешение возникают две различные коды ошибки которые могут быть уместны.

* [HTTP 401 Unauthorized][http401]
* [HTTP 403 Permission Denied][http403]

HTTP 401 responses must always include a `WWW-Authenticate` header, that instructs the client how to authenticate.  HTTP 403 responses do not include the `WWW-Authenticate` header.
Ответ HTTP 401 должен постоянно включать `WWW-Authenticate` хедер, который сообщит клиенту как авторизоваться. Ответ HTTP 403 не включает такого хедера.

The kind of response that will be used depends on the authentication scheme.  Although multiple authentication schemes may be in use, only one scheme may be used to determine the type of response.  **The first authentication class set on the view is used when determining the type of response**.
Тип ответа зависит от авторизационной схемы. Хотя множество авторизационных схем могут быть использованы, только одна схема может быть использована чтобы определить тип ответа. **Первый авторизационный класс вьюшки используется когда определяется тип ответа**

Note that when a request may successfully authenticate, but still be denied permission to perform the request, in which case a `403 Permission Denied` response will always be used, regardless of the authentication scheme.
Заметьте, что когда запрос может успешно аторизован, но все еще получает отказ в производстве запроса, в таком случае ответ `403 Permission Denied` несмотря на авторизационную схему.

## Apache mod_wsgi specific configuration 

Note that if deploying to [Apache using mod_wsgi][mod_wsgi_official], the authorization header is not passed through to a WSGI application by default, as it is assumed that authentication will be handled by Apache, rather than at an application level.

If you are deploying to Apache, and using any non-session based authentication, you will need to explicitly configure mod_wsgi to pass the required headers through to the application.  This can be done by specifying the `WSGIPassAuthorization` directive in the appropriate context and setting it to `'On'`.

    # this can go in either server config, virtual host, directory or .htaccess
    WSGIPassAuthorization On

---

# API Reference | Справочник API

## BasicAuthentication | Класс BaseAuthentication

This authentication scheme uses [HTTP Basic Authentication][basicauth], signed against a user's username and password.  Basic authentication is generally only appropriate for testing.
Эта авторизационная схема использует [HTTP Basic Authentication][basicauth], подписаная паролем и именем пользователя.  Базовая авторизация в общемто подходит только для тестирования.

If successfully authenticated, `BasicAuthentication` provides the following credentials.
Если успешно авторизовался, предоставляет следующие полномочия.

* `request.user` will be a Django `User` instance. будет экземплят класс User
* `request.auth` will be `None`. будет None

Unauthenticated responses that are denied permission will result in an `HTTP 401 Unauthorized` response with an appropriate WWW-Authenticate header.  For example:
Неавторизованные ответы которым отказано в разрешении будут с HTTP 401 ответом с соответсвующим хедером. Для примера:

    WWW-Authenticate: Basic realm="api"

**Note:** If you use `BasicAuthentication` in production you must ensure that your API is only available over `https`.  You should also ensure that your API clients will always re-request the username and password at login, and will never store those details to persistent storage.
**Примечание:** Если вы используете `BasicAuthentication` в продакшене, вы должны убедиться что ваш API доступен только по `https`. Вам также следует убедится что вашы API клиенты будут всегда перезапрашивать юзернайм и пароль и никогда не сохранять эти детали для постоянного хранения.

## TokenAuthentication | Класс TokenAuthentication

This authentication scheme uses a simple token-based HTTP Authentication scheme.  Token authentication is appropriate for client-server setups, such as native desktop and mobile clients.
Эта авторизационная схема ипользует простой токен. Авторизация с помощью токена подходит для клиент-серверных установок, таких как декстопы и мобильные клиенты.

To use the `TokenAuthentication` scheme you'll need to [configure the authentication classes](#setting-the-authentication-scheme) to include `TokenAuthentication`, and additionally include `rest_framework.authtoken` in your `INSTALLED_APPS` setting:
Чтобы использовать `TokenAuthentication` схему, вам понадобится [configure the authentication classes](#setting-the-authentication-scheme) чтобы включить `TokenAuthentication`, и дополнительно включить `rest_framework.authtoken` в ваши `INSTALLED_APPS` настройки

    INSTALLED_APPS = (
        ...
        'rest_framework.authtoken'
    )

---

**Note:** Make sure to run `manage.py syncdb` after changing your settings. The `rest_framework.authtoken` app provides both Django (from v1.7) and South database migrations. See [Schema migrations](#schema-migrations) below.
**Примечание:** Убедитесь что запустили `manage.py syncdb` после изменений ваших настроек. `rest_framework.authtoken` предоставляет как для Django (с версии 1.7) так и South миграцию. Смотри [Schema migrations](#schema-migrations) ниже.

---


You'll also need to create tokens for your users.
Вам нужно будет создать токен для ваших пользователей.

    from rest_framework.authtoken.models import Token

    token = Token.objects.create(user=...)
    print token.key

For clients to authenticate, the token key should be included in the `Authorization` HTTP header.  The key should be prefixed by the string literal "Token", with whitespace separating the two strings.  For example:
Чтобы клиентам авторизоваться, ключ токена следует включить в хедер `Authorization`. Перед ключем должен установлет префикс "Token" с пробелом

    Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b

If successfully authenticated, `TokenAuthentication` provides the following credentials.
Если успешно авторизовался, `TokenAuthentication` предоставить следующие полномочия.

* `request.user` will be a Django `User` instance.
* `request.auth` will be a `rest_framework.authtoken.models.BasicToken` instance.

Unauthenticated responses that are denied permission will result in an `HTTP 401 Unauthorized` response with an appropriate WWW-Authenticate header.  For example:
Неавторизованные ответы которым отказано в разрешении будут с HTTP 401 ответом с соответсвующим хедером. Для примера:

    WWW-Authenticate: Token

The `curl` command line tool may be useful for testing token authenticated APIs.  For example:
Можно использовать `curl` для тестирования авторизации. Для примера:

    curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'

---

**Note:** If you use `TokenAuthentication` in production you must ensure that your API is only available over `https`.
**Примечание** Если вы используете класс `TokenAuthentication` в продакшене вы должны убедиться что ваш API только доступен по `https`. 

---

#### Generating Tokens | Генерация токенов

If you want every user to have an automatically generated Token, you can simply catch the User's `post_save` signal.
Если вы хотите чтобы каждый пользователь получал сгенерированый токен, вы можете просто ловить сигнал `post_save`.

    from django.conf import settings
    from django.db.models.signals import post_save
    from django.dispatch import receiver
    from rest_framework.authtoken.models import Token

    @receiver(post_save, sender=settings.AUTH_USER_MODEL)
    def create_auth_token(sender, instance=None, created=False, **kwargs):
        if created:
            Token.objects.create(user=instance)

Note that you'll want to ensure you place this code snippet in an installed `models.py` module, or some other location that will be imported by Django on startup.
Заметьте что вы зохотите убедится что вы положили этот фрагмент кода в установленый `models.py` или друго место которое будет импортировано Джангой при старте.

If you've already created some users, you can generate tokens for all existing users like this:
Если вы уже создали некоторых пользоватей, вы можете сгенерировать токен для всех существующих пользователей так:

    from django.contrib.auth.models import User
    from rest_framework.authtoken.models import Token

    for user in User.objects.all():
        Token.objects.get_or_create(user=user)

When using `TokenAuthentication`, you may want to provide a mechanism for clients to obtain a token given the username and password.  REST framework provides a built-in view to provide this behavior.  To use it, add the `obtain_auth_token` view to your URLconf:
Когда используете `TokenAuthentication` вы возможно захотите предоставить механизм для клиентов получить токен по юзернайму и паролю. REST framework предоставляет встроеную вьюшку для этого. Чтобы использовать ее, добавте `obtain_auth_token` в ваш URLconf:

    from rest_framework.authtoken import views
    urlpatterns += [
        url(r'^api-token-auth/', views.obtain_auth_token)
    ]

Note that the URL part of the pattern can be whatever you want to use.
Заметьте что URL часть паттерная который вы можете не использовать.

The `obtain_auth_token` view will return a JSON response when valid `username` and `password` fields are POSTed to the view using form data or JSON:
`obtain_auth_token` будет возвращать JSON ответ куда отправять валидные имяпользователя и пароль в POST запросе.

    { 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }

Note that the default `obtain_auth_token` view explicitly uses JSON requests and responses, rather than using default renderer and parser classes in your settings.  If you need a customized version of the `obtain_auth_token` view, you can do so by overriding the `ObtainAuthToken` view class, and using that in your url conf instead.
Заметьте что по умолчанию `obtain_auth_token` явно используют JSON запросы и ответы, вместо использования по умолчанию визуализации и парсинга классов в ваших настройках. Если вам нужна своя версия `obtain_auth_token`, вы можете переопределить `ObtainAuthToken` класс и использовать его в ваших url.

#### Schema migrations | Схема миграции

The `rest_framework.authtoken` app includes both Django native migrations (for Django versions >1.7) and South migrations (for Django versions <1.7) that will create the authtoken table.
Приложение `rest_framework.authtoken` включает как Джанго миграцию, так и South миграцию которые создадут таблицу с токенами.

----

**Note**: From REST Framework v2.4.0 using South with Django <1.7 requires upgrading South v1.0+
**Примечание**: Начиная с версии 2.4.0 необходимо обновление South до верссии 1.0+

----


If you're using a [custom user model][custom-user-model] you'll need to make sure that any initial migration that creates the user table runs before the authtoken table is created.
Если вы используете [свою модель пользователей][custom-user-model] вам нужно будет убедиться, что каждая новая миграция которая создает таблицу с пользователями запускается до создания таблицы с токенами.

You can do so by inserting a `needed_by` attribute in your user migration:
Для этого вы можете вставить атребут `needed_by` внутрь вашей миграции:

    class Migration:

        needed_by = (
            ('authtoken', '0001_initial'),
        )

        def forwards(self):
            ...

For more details, see the [south documentation on dependencies][south-dependencies].
Детали смотрите в [документации к South][south-dependencies].

Also note that if you're using a `post_save` signal to create tokens, then the first time you create the database tables, you'll need to ensure any migrations are run prior to creating any superusers.  For example:
Также помните, что если используете сигнал `post_save` чтобы создать токен, вам нужно убедиться, что любая миграция запускается до создания супер пользователя во время первого создания таблиц в базе. Для примера:

    python manage.py syncdb --noinput  # Won't create a superuser just yet, due to `--noinput`.
    python manage.py migrate
    python manage.py createsuperuser

## SessionAuthentication | Класс SessionAuthentication

This authentication scheme uses Django's default session backend for authentication.  Session authentication is appropriate for AJAX clients that are running in the same session context as your website.
Такая авторизационная схема используется Джанго по умолчанию. Авторизация сессиями подходит для AJAX клиентов которые запускают одинаковый сессионный контекст как ваш сайт

If successfully authenticated, `SessionAuthentication` provides the following credentials.
Если успешно авторизоваться, предоставляются следующие полномочия:

* `request.user` will be a Django `User` instance. будет экземпляр Django User
* `request.auth` will be `None`. будет None

Unauthenticated responses that are denied permission will result in an `HTTP 403 Forbidden` response.
Неавторизованные ответы которым был отказано в разрешение будут возвращаться с ответом `HTTP 403 Forbidden`.

If you're using an AJAX style API with SessionAuthentication, you'll need to make sure you include a valid CSRF token for any "unsafe" HTTP method calls, such as `PUT`, `PATCH`, `POST` or `DELETE` requests.  See the [Django CSRF documentation][csrf-ajax] for more details.
Если вы используете AJAX стиль с SessionAuthentucation, вам нужно будет убедиться что CSRF токен включен для любого "не безопасного" HTTP метода, такого как `PUT`, `PATH`, `POST` или `DELETE`. Смотрите [CSRF Django документацию][csrf-ajax] для подробностей.

**Warning**: Always use Django's standard login view when creating login pages. This will ensure your login views are properly protected.
**Внимание**: Всегда используйте стандартный Джанго логин когда создаете страницу с логином. Это защитит вас.

CSRF validation in REST framework works slightly differently to standard Django due to the need to support both session and non-session based authentication to the same views. This means that only authenticated requests require CSRF tokens, and anonymous requests may be sent without CSRF tokens. This behaviour is not suitable for login views, which should always have CSRF validation applied.
CSRF валидация в REST работает немного по другому чем в Джанго, по причине необходимости поддержки как сессионного так и несессионного метода авторизации в одной вьюшке. Это значит, что только авторизационный запрос требудет CSRF токен, и анонимные запросы могут быть отправилены без CSRF токена. Это поведение не соответсвует логин вьюшке, которое должно всегда иметь CSRF.

# Custom authentication | Кастомные авторизации

To implement a custom authentication scheme, subclass `BaseAuthentication` and override the `.authenticate(self, request)` method.  The method should return a two-tuple of `(user, auth)` if authentication succeeds, or `None` otherwise.
Чтобы осуществить кастомную авторизации, нужно наследоваться от `BaseAuthentication` и переопределить  метод `.authenticate(self, request)` . Метод должен возвращать тюпла вида `(user, auth)` если авторизация успешна, или `None`.

In some circumstances instead of returning `None`, you may want to raise an `AuthenticationFailed` exception from the `.authenticate()` method.
В некоторых обстоятельствах вместо возвращения `None`, вы можете получить исключение `AuthenticationFailed` из метода `.authenticate()` .

Typically the approach you should take is:
Обычно вас следует придерживаться следующих подходов:

* If authentication is not attempted, return `None`.  Any other authentication schemes also in use will still be checked.
* Если авторизация не пробовалась, вернет `None`. Любые другие авторизационные схемы также будут проверсятбся.
* If authentication is attempted but fails, raise a `AuthenticationFailed` exception.  An error response will be returned immediately, regardless of any permissions checks, and without checking any other authentication schemes.
* Если авторизация пробовалась но провалилась, сработает исключение `AuthenticationFailed`. Ошибочный ответ будет возвращен немедленно, несмотря на любые проверки разрешений, и без проверки любых других авторизационных схем.

You *may* also override the `.authenticate_header(self, request)` method.  If implemented, it should return a string that will be used as the value of the `WWW-Authenticate` header in a `HTTP 401 Unauthorized` response.
Вы *можете* также переопределить метод `.authenticate_header(self, request)`. Если он есть, он должен вернуть строку которая будет использоваться как значение хедера `WWW-Authenticate` в ответе `HTTP 401 Unauthorized`.

If the `.authenticate_header()` method is not overridden, the authentication scheme will return `HTTP 403 Forbidden` responses when an unauthenticated request is denied access.
Если метод `.authenticate_header()` не переопределен, авторизационная схема будет возвращать `HTTP 403 Forbidden` когда неавторизованному запросу откжут в доступе.

## Example | Пример

The following example will authenticate any incoming request as the user given by the username in a custom request header named 'X_USERNAME'.
Следующие пример будет авторизовывать любой входящий запрос где будет прописано в хедере имя пользователя как 'X_USERNAME'.

	from django.contrib.auth.models import User
    from rest_framework import authentication
    from rest_framework import exceptions

    class ExampleAuthentication(authentication.BaseAuthentication):
        def authenticate(self, request):
            username = request.META.get('X_USERNAME')
            if not username:
                return None

            try:
                user = User.objects.get(username=username)
            except User.DoesNotExist:
                raise exceptions.AuthenticationFailed('No such user')

            return (user, None)

---

# Third party packages | Пакеты третих лиц

The following third party packages are also available.
Следующие пакет также доступны.

## Django OAuth Toolkit

The [Django OAuth Toolkit][django-oauth-toolkit] package provides OAuth 2.0 support, and works with Python 2.7 and Python 3.3+. The package is maintained by [Evonove][evonove] and uses the excellent [OAuthLib][oauthlib].  The package is well documented, and well supported and is currently our **recommended package for OAuth 2.0 support**.
Пакет [Django OAuth Toolkit][django-oauth-toolkit] поддерживает OAuth 2.0 и работает с Python 2.7 и Python 3.3+. Пакет поддерживается [Evonove][evonove] и использует замечательную библиотеку [OAuthLib][oauthlib]. Пакет хорошо документирован и хорошо поддерживается и это наш **рекомендованный пакет для OAuth 2.0**.

#### Installation & configuration | Установка и конфигурация

Install using `pip`. Установите используя `pip`.

    pip install django-oauth-toolkit

Add the package to your `INSTALLED_APPS` and modify your REST framework settings.
Добавьте пакет к вашим `INSTALLED_APPS` и измените настройки REST framework.

    INSTALLED_APPS = (
        ...
        'oauth2_provider',
    )

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': (
            'oauth2_provider.ext.rest_framework.OAuth2Authentication',
        )
    }

For more details see the [Django REST framework - Getting started][django-oauth-toolkit-getting-started] documentation.
Детали смотрите на [Django REST framework - Getting started][django-oauth-toolkit-getting-started]

## Django REST framework OAuth

The [Django REST framework OAuth][django-rest-framework-oauth] package provides both OAuth1 and OAuth2 support for REST framework.
Пакет [Django REST framework OAuth][django-rest-framework-oauth] поддерживает как OAuth1 так и OAuth2.

This package was previously included directly in REST framework but is now supported and maintained as a third party package.
Этот пакет был включен в REST framework но сейчас поддерживается и разрабатывается как продукт третьей стороны.

#### Installation & configuration | Установка и конфигурация

Install the package using `pip`. Установите с помощью `pip`

    pip install djangorestframework-oauth

For details on configuration and usage see the Django REST framework OAuth documentation for [authentication][django-rest-framework-oauth-authentication] and [permissions][django-rest-framework-oauth-permissions].
Про детали конфигурирования и использования смотрите Django REST framework OAuth документацию [authentication][django-rest-framework-oauth-authentication] и [permissions][django-rest-framework-oauth-permissions].

## Digest Authentication

HTTP digest authentication is a widely implemented scheme that was intended to replace HTTP basic authentication, and which provides a simple encrypted authentication mechanism. [Juan Riaza][juanriaza] maintains the [djangorestframework-digestauth][djangorestframework-digestauth] package which provides HTTP digest authentication support for REST framework.
HTTP digest authentication широко применяемая схема которая предназначена чтобы заменить базовую HTTP авторизацию и которая предоставляет простой шифрующий авторизационный механизм. [Juan Riaza][juanriaza] поддерживает пакет [djangorestframework-digestauth][djangorestframework-digestauth] который поддерживается REST framework.

## Django OAuth2 Consumer

The [Django OAuth2 Consumer][doac] library from [Rediker Software][rediker] is another package that provides [OAuth 2.0 support for REST framework][doac-rest-framework].  The package includes token scoping permissions on tokens, which allows finer-grained access to your API.
Библиотека [Django OAuth2 Consumer][doac] от [Rediker Software][rediker] - это еще один пакет для [OAuth 2.0][doac-rest-framework].

## JSON Web Token Authentication

JSON Web Token is a fairly new standard which can be used for token-based authentication. Unlike the built-in TokenAuthentication scheme, JWT Authentication doesn't need to use a database to validate a token. [Blimp][blimp] maintains the [djangorestframework-jwt][djangorestframework-jwt] package which provides a JWT Authentication class as well as a mechanism for clients to obtain a JWT given the username and password.
JWT - относительно новый стандарт который может быть использован для токен авторизации. В отличии от встроенной токен схемы, JWT авторизации не нужна база данных для валидации токена. Разработкой и поддержкой занимается [Blimp][blimp] с пакетом [djangorestframework-jwt][djangorestframework-jwt] который предоставляет JWT Authentication класс как механизм для клиентов чтобы получать JWT по имени пользователя и паролю.

## Hawk HTTP Authentication

The [HawkREST][hawkrest] library builds on the [Mohawk][mohawk] library to let you work with [Hawk][hawk] signed requests and responses in your API. [Hawk][hawk] lets two parties securely communicate with each other using messages signed by a shared key. It is based on [HTTP MAC access authentication][mac] (which was based on parts of [OAuth 1.0][oauth-1.0a]).
Библиотека [HawkREST][hawkrest] построена на библиотеке [Mohawk][mohawk] позволя вам работать с [Hawk][hawk] подписывая запросу и ответы вашего API. [Hawk][hawk] позволяет двум частям безопасно комуницировать с друг другом используя подписи с общим ключем. Это базируется на [HTTP MAC access authentication][mac] (который базируется от части на [OAuth 1.0][oauth-1.0a]).

## HTTP Signature Authentication 

HTTP Signature (currently a [IETF draft][http-signature-ietf-draft]) provides a way to achieve origin authentication and message integrity for HTTP messages. Similar to [Amazon's HTTP Signature scheme][amazon-http-signature], used by many of its services, it permits stateless, per-request authentication. [Elvio Toccalino][etoccalino] maintains the [djangorestframework-httpsignature][djangorestframework-httpsignature] package which provides an easy to use HTTP Signature Authentication mechanism.
Подпись HTTP (сейчас [IETF draft][http-signature-ietf-draft]) предоставляет способ достижения первоначальной авторизации и интеграцию сообщейний для HTTP сообщений. Подобно [Amazon's HTTP Signature scheme][amazon-http-signature], используетмя во многих своих сервисах, это разрешение без гражданства, для каждого запроса авторизации. [Elvio Toccalino][etoccalino] поддерживает  пакет [djangorestframework-httpsignature][djangorestframework-httpsignature] для простого использования механизма HTTP подписи. 

## Djoser 

[Djoser][djoser] library provides a set of views to handle basic actions such as registration, login, logout, password reset and account activation. The package works with a custom user model and it uses token based authentication. This is a ready to use REST implementation of Django authentication system.
Библиотека [Djoser][djoser] предоставляет набор вьюшек для обработки базовых действий таких как регистрация, логирование, выход из системы, сброс пароля и активация аккаунта. Паке работает с моделями кастомных пользователй и использует авторизацию основаную на токенах. Она готова использовать REST авторизационной системы Джанго.

## django-rest-auth

[Django-rest-auth][django-rest-auth] library provides a set of REST API endpoints for registration, authentication (including social media authentication), password reset, retrieve and update user details, etc. By having these API endpoints, your client apps such as AngularJS, iOS, Android, and others can communicate to your Django backend site independently via REST APIs for user management. 
Библиотека [Django-rest-auth][django-rest-auth] предоставляет набор инструментов для REST API: регистрация, авторизация (включая авторизацию через социальные сети), сброс пароля, получение и обновление пользовательских деталей и тд. С этими инструментами, ваше клиентское приложение как AngularJS, iOS, Android и другие смогут взаимодействовать с Джанго бэкендом независимо по средством REST APIs. 

[cite]: http://jacobian.org/writing/rest-worst-practices/
[http401]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.2
[http403]: http://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html#sec10.4.4
[basicauth]: http://tools.ietf.org/html/rfc2617
[oauth]: http://oauth.net/2/
[permission]: permissions.md
[throttling]: throttling.md
[csrf-ajax]: https://docs.djangoproject.com/en/dev/ref/csrf/#ajax
[mod_wsgi_official]: http://code.google.com/p/modwsgi/wiki/ConfigurationDirectives#WSGIPassAuthorization
[custom-user-model]: https://docs.djangoproject.com/en/dev/topics/auth/customizing/#specifying-a-custom-user-model
[south-dependencies]: http://south.readthedocs.org/en/latest/dependencies.html
[django-oauth-toolkit-getting-started]: https://django-oauth-toolkit.readthedocs.org/en/latest/rest-framework/getting_started.html
[django-rest-framework-oauth]: http://jpadilla.github.io/django-rest-framework-oauth/
[django-rest-framework-oauth-authentication]: http://jpadilla.github.io/django-rest-framework-oauth/authentication/
[django-rest-framework-oauth-permissions]: http://jpadilla.github.io/django-rest-framework-oauth/permissions/
[juanriaza]: https://github.com/juanriaza
[djangorestframework-digestauth]: https://github.com/juanriaza/django-rest-framework-digestauth
[oauth-1.0a]: http://oauth.net/core/1.0a
[django-oauth-plus]: http://code.larlet.fr/django-oauth-plus
[django-oauth2-provider]: https://github.com/caffeinehit/django-oauth2-provider
[django-oauth2-provider-docs]: https://django-oauth2-provider.readthedocs.org/en/latest/
[rfc6749]: http://tools.ietf.org/html/rfc6749
[django-oauth-toolkit]: https://github.com/evonove/django-oauth-toolkit
[evonove]: https://github.com/evonove/
[oauthlib]: https://github.com/idan/oauthlib
[doac]: https://github.com/Rediker-Software/doac
[rediker]: https://github.com/Rediker-Software
[doac-rest-framework]: https://github.com/Rediker-Software/doac/blob/master/docs/integrations.md#
[blimp]: https://github.com/GetBlimp
[djangorestframework-jwt]: https://github.com/GetBlimp/django-rest-framework-jwt
[etoccalino]: https://github.com/etoccalino/
[djangorestframework-httpsignature]: https://github.com/etoccalino/django-rest-framework-httpsignature
[amazon-http-signature]: http://docs.aws.amazon.com/general/latest/gr/signature-version-4.html
[http-signature-ietf-draft]: https://datatracker.ietf.org/doc/draft-cavage-http-signatures/
[hawkrest]: http://hawkrest.readthedocs.org/en/latest/
[hawk]: https://github.com/hueniverse/hawk
[mohawk]: http://mohawk.readthedocs.org/en/latest/
[mac]: http://tools.ietf.org/html/draft-hammer-oauth-v2-mac-token-05
[djoser]: https://github.com/sunscrapers/djoser
[django-rest-auth]: https://github.com/Tivix/django-rest-auth
