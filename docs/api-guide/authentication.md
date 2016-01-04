source: authentication.py

# Аутентификация

> Аутентификация должна быть подключаемой
>
> &mdash; Jacob Kaplan-Moss, ["REST worst practices"][cite]

Аутентификация это механизм связывания входящего запроса с набором определенных полномочий или параметров, таких как пользователь от которого пришел запрос, или токен которым подписан этот запрос. [Разрешения][permission] и [ограничения][throttling] прав затем могут использовать эти параметры чтобы определить разрешения для запроса.

REST framework предоставляет несколько аутентификационных схем из коробки, и также позволяет вам использовать свои схемы.

Аутентификация всегда запускается в самом начале view, до того как происходить проверка разрешений и ограничений, и до того как начнется выполняться другой код.

Свойство `request.user` обычно установлено в экземпляре `contrib.auth` класса `User` 

Свойство `request.auth` используется для любой дополнительной аутентификационной информации, для примера, оно может использовать для предоставления аутификационного токена которым будет подписан запрос.

---

**Примечание** Не забывайте, что **аутентификация сама по себе не позволяет или запрещает входящие запросы**, она просто определяет полномочия которые даны запросу.
Для информации о том как установить политику разрешений для вашего API пожалуйста смотрите [документацию разрешений][permission].
---

## Как аутентификация определяется

Аутентификационные схемы всегда определяются как список классов. REST fremework будет пытаться авторизоваться к каждому классу в списке, и установит `request.user` и `request.auth` используя возвращаемое значение первого класса который успешно авторизуется.

Если не один класс не авторизуется, `request.user` будет установлен в экземпляр `django.contrib.auth.models.AnonymousUser`, и `request.auth` будет установлен как `None`.

Значение `request.user` и `request.auth` для неавторизованых запросов может быть изменено с помощью `UNAUTHENTICATED_USER` и `UNAUTHENTICATED_TOKEN` настроек.

## Настройки аутентификационной схемы

По умолчанию аутентификационная схема может быть установленна глобально, в `DEFAULT_AUTHENTICATION_CLASSES` файла settings.py

    REST_FRAMEWORK = {
        'DEFAULT_AUTHENTICATION_CLASSES': (
            'rest_framework.authentication.BasicAuthentication',
            'rest_framework.authentication.SessionAuthentication',
        )
    }

Также вы можете установить аутентификационную схему для каждого view отдельно или для view наследованных от `APIView`

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

## 401 и 403 ответы

Когда неавторизованному запросу отказано в разрешение возникает два различных кода ошибки, которые могут быть уместны.

* [HTTP 401 Unauthorized][http401]
* [HTTP 403 Permission Denied][http403]

Ответ HTTP 401 должен постоянно включать хедер `WWW-Authenticate`, который сообщит клиенту как авторизоваться. Ответ HTTP 403 не включает такого хедера.

Тип ответа зависит от аутентификационной схемы. Хотя множество аутентификационных схем могут быть использованы, только одна схема может быть использована чтобы определить тип ответа. **Первый аутентификационный класс view используется чтобы определить тип ответа**

Заметьте, что запрос может успешно аторизоваться, но получить отказ для продолжения запроса, в таком случае будет ответ `403 Permission Denied` несмотря на аутентификационную схему.

## Конфигурация Apache mod_wsgi

Имейте в виду, если разворачиваете на [Apache используя mod_wsgi][mod_wsgi_official], аутентификационный хедер не дойдет до WSGI приложения по умолчанию, так как предпролагается, что аутентификацию обрабатывает Apache и до уровня приложение ничего не дойдет.

Если разворачиваетесь на Apache и используете любую не сессионную аутентификацию, вам нужно явно конфигурировать mod_wsgi чтобы передать необходимый хедер до приложения. Для этого определите директиву `WSGIPassAuthorization` в необходимом контексте и установите ее в положение `'On'`.

    # this can go in either server config, virtual host, directory or .htaccess
    WSGIPassAuthorization On

---

# Справочник API

## Класс BaseAuthentication

Эта аутентификационная схема использует [HTTP Basic Authentication][basicauth], подписаная паролем и именем пользователя. Базовая аутентификация в общем-то подходит только для тестирования.

Если успешно авторизовался, предоставляет следующие полномочия.

* `request.user` будет экземпляр класса User
* `request.auth` будет None

Неавторизованные ответы которым отказано в разрешении будут с HTTP 401 ответом с соответсвующим хедером. Для примера:

    WWW-Authenticate: Basic realm="api"

**Примечание:** Если вы используете `BasicAuthentication` в продакшене, вы должны убедиться что ваш API доступен только по `https`. Вам также следует убедится что вашы API клиенты будут всегда перезапрашивать юзернайм и пароль и никогда не сохранять эти детали для постоянного хранения.

## Класс TokenAuthentication

Эта аутентификационная схема ипользует простой токен. Аутентификация с помощью токена подходит для клиент-серверных установок, таких как настольные компьютеры и мобильные клиенты.

Чтобы использовать `TokenAuthentication` схему, вам понадобится [сконфигурировать аутентификационные классы](#setting-the-authentication-scheme) чтобы включить `TokenAuthentication`, и дополнительно включить `rest_framework.authtoken` в ваших настройках `INSTALLED_APPS`:

    INSTALLED_APPS = (
        ...
        'rest_framework.authtoken'
    )

---

**Примечание:** Убедитесь что запустили `manage.py syncdb` после изменений ваших настроек. `rest_framework.authtoken` предоставляет миграцию как для Django (с версии 1.7) так и South. Смотри [миграционные схемы](#schema-migrations) ниже.

---


Вам нужно будет создать токены для ваших пользователей.

    from rest_framework.authtoken.models import Token

    token = Token.objects.create(user=...)
    print token.key

Чтобы клиентам авторизоваться, ключ токена следует включить в хедер `Authorization`. Перед ключем необходимо установить префикс "Token" с пробелом

    Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b

Если успешно авторизовался, `TokenAuthentication` предоставить следующие полномочия.

* `request.user` will be a Django `User` instance.
* `request.auth` will be a `rest_framework.authtoken.models.BasicToken` instance.

Неавторизованные ответы которым отказано в разрешении будут с кодом ответа HTTP 401 и хедером WWW-Authenticate. Для примера:

    WWW-Authenticate: Token

Можно использовать `curl` для тестирования авторизации. Для примера:

    curl -X GET http://127.0.0.1:8000/api/example/ -H 'Authorization: Token 9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b'

---

**Примечание** Если вы используете класс `TokenAuthentication` в продакшене вы должны убедиться что ваш API только доступен по `https`. 

---

#### Генерация токенов

Если вы хотите чтобы каждый пользователь получал сгенерированый токен, вы можете просто ловить сигнал `post_save`.

    from django.conf import settings
    from django.db.models.signals import post_save
    from django.dispatch import receiver
    from rest_framework.authtoken.models import Token

    @receiver(post_save, sender=settings.AUTH_USER_MODEL)
    def create_auth_token(sender, instance=None, created=False, **kwargs):
        if created:
            Token.objects.create(user=instance)

Заметьте что вы зохотите убедится что вы положили этот фрагмент кода в установленый `models.py` или друго место которое будет импортировано Джангой при старте.

Если вы уже создали некоторых пользоватей, вы можете сгенерировать токен для всех существующих пользователей так:

    from django.contrib.auth.models import User
    from rest_framework.authtoken.models import Token

    for user in User.objects.all():
        Token.objects.get_or_create(user=user)

Когда используете `TokenAuthentication` вы возможно захотите предоставить клиентам механизм получения токена по имени пользователя и паролю. REST framework предоставляет встроенную функцию для этого. Чтобы использовать ее, добавьте `obtain_auth_token` в ваш URLconf:

    from rest_framework.authtoken import views
    urlpatterns += [
        url(r'^api-token-auth/', views.obtain_auth_token)
    ]

Имейте в виду, что URL часть паттерная который вы можете не использовать.

`obtain_auth_token` будет возвращать JSON ответ когда получит валидное имя пользователя и пароль в POST запросе.

    { 'token' : '9944b09199c62bcf9418ad846dd0e4bbdfc6ee4b' }

Имейте в виду, что по умолчанию `obtain_auth_token` явно использует JSON запросы и ответы, вместо использования по умолчанию визуализации и парсинга классов в ваших настройках. Если вам нужна своя версия `obtain_auth_token`, вы можете переопределить `ObtainAuthToken` класс и использовать его в ваших конфигурация url.

#### Схема миграции

Приложение `rest_framework.authtoken` включает как Джанго миграцию, так и South миграцию которые создадут таблицу с токенами.

----

**Примечание**: Начиная с версии 2.4.0 необходимо обновление South до верссии 1.0+

----


Если вы используете [свою модель пользователей][custom-user-model] вам нужно будет убедиться, что каждая новая миграция которая создает таблицу с пользователями запускается до создания таблицы с токенами.

Для этого вы можете вставить атрибут `needed_by` внутрь вашей миграции:

    class Migration:

        needed_by = (
            ('authtoken', '0001_initial'),
        )

        def forwards(self):
            ...

Детали смотрите в [документации к South][south-dependencies].

Также помните, что если используете сигнал `post_save` чтобы создать токен, вам нужно убедиться, что любая миграция запускается до создания супер пользователя во время первого создания таблиц в базе. Для примера:

    python manage.py syncdb --noinput  # Won't create a superuser just yet, due to `--noinput`.
    python manage.py migrate
    python manage.py createsuperuser

## Класс SessionAuthentication

Такая аутентификационная схема используется Джанго по умолчанию. Авторизация сессиями подходит для AJAX клиентов которые запускают одинаковый сессионный контекст как ваш сайт

Если успешно авторизовался, предоставляются следующие полномочия:

* `request.user` будет экземпляр Django User
* `request.auth` будет None

Неавторизованные ответы которым было отказано в разрешение будут возвращаться с ответом `HTTP 403 Forbidden`.

Если вы используете AJAX стиль с SessionAuthentucation, вам нужно будет убедиться что CSRF токен включен для любого "не безопасного" HTTP метода, такого как `PUT`, `PATH`, `POST` или `DELETE`. Смотрите [CSRF Django документацию][csrf-ajax] для подробностей.

**Внимание**: Всегда используйте стандартный Джанго логин когда создаете страницу с логином. Это защитит вас.

CSRF валидация в REST framework работает немного по другому чем в Джанго, по причине необходимости поддержки как сессионного так и несессионного метода авторизации в одной view. Это значит, что только аутентификационный запрос требудет CSRF токен, и анонимные запросы могут быть отправилены без CSRF токена. Это поведение не соответсвует логин view, которая должно всегда иметь CSRF.

# Кастомные аутентификации

Чтобы осуществить кастомную аутентификацию, нужно наследоваться от `BaseAuthentication` и переопределить  метод `.authenticate(self, request)` . Метод должен возвращать кортеж вида `(user, auth)` если авторизация успешна, или `None`.

В некоторых обстоятельствах вместо возвращения `None`, вы можете получить исключение `AuthenticationFailed` из метода `.authenticate()` .

Обычно вам следует придерживаться следующих подходов:

* Если аутентификация не случилась, вернет `None`. Любые другие аутентификационные схемы также будут проверяться.
* Если авторизация случилась но провалилась, сработает исключение `AuthenticationFailed`. Ошибочный ответ будет возвращен немедленно, несмотря на любые проверки разрешений, и без проверки любых других авторизационных схем.

Вы *можете* также переопределить метод `.authenticate_header(self, request)`. Если он есть, он должен вернуть строку которая будет использоваться как значение хедера `WWW-Authenticate` в ответе `HTTP 401 Unauthorized`.

Если метод `.authenticate_header()` не переопределен, аутентификационная схема будет возвращать `HTTP 403 Forbidden` когда неавторизованному запросу откажут в доступе.

## Пример

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

# Пакеты третих лиц

Следующие пакет также доступны.

## Django OAuth Toolkit

Пакет [Django OAuth Toolkit][django-oauth-toolkit] поддерживает OAuth 2.0 и работает с Python 2.7 и Python 3.3+. Пакет поддерживается [Evonove][evonove] и использует замечательную библиотеку [OAuthLib][oauthlib]. Пакет хорошо документирован и хорошо поддерживается и это наш **рекомендованный пакет для OAuth 2.0**.

#### Установка и конфигурация

Установите используя `pip`.

    pip install django-oauth-toolkit

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

Детали смотрите на [Django REST framework - Getting started][django-oauth-toolkit-getting-started]

## Django REST framework OAuth

Пакет [Django REST framework OAuth][django-rest-framework-oauth] поддерживает как OAuth1 так и OAuth2.

Этот пакет был включен в REST framework но сейчас поддерживается и разрабатывается как продукт третьей стороны.

#### Установка и конфигурация

Установите с помощью `pip`

    pip install djangorestframework-oauth

Про детали конфигурирования и использования смотрите Django REST framework OAuth документацию [authentication][django-rest-framework-oauth-authentication] и [permissions][django-rest-framework-oauth-permissions].

## Digest Authentication

HTTP digest authentication широко применяемая схема которая предназначена чтобы заменить базовую HTTP авторизацию и которая предоставляет простой шифрующий авторизационный механизм. [Juan Riaza][juanriaza] поддерживает пакет [djangorestframework-digestauth][djangorestframework-digestauth] который поддерживается REST framework.

## Django OAuth2 Consumer

Библиотека [Django OAuth2 Consumer][doac] от [Rediker Software][rediker] - это еще один пакет для [OAuth 2.0][doac-rest-framework].

## JSON Web Token Authentication

JWT - относительно новый стандарт который может быть использован для токен авторизации. В отличии от встроенной токен схемы, JWT авторизации не нужна база данных для валидации токена. Разработкой и поддержкой занимается [Blimp][blimp] с пакетом [djangorestframework-jwt][djangorestframework-jwt] который предоставляет JWT Authentication класс как механизм получения JWT токена по имени пользователя и паролю.

## Hawk HTTP Authentication

Библиотека [HawkREST][hawkrest] построена на библиотеке [Mohawk][mohawk] позволяет вам работать с [Hawk][hawk] подписывая запросу и ответы вашего API. [Hawk][hawk] позволяет двум частям безопасно комуницировать с друг другом используя подписи с общим ключем. Это базируется на [HTTP MAC access authentication][mac] (который базируется от части на [OAuth 1.0][oauth-1.0a]).

## HTTP Signature Authentication 

Подпись HTTP (сейчас [IETF draft][http-signature-ietf-draft]) предоставляет способ достижения первоначальной авторизации и интеграцию сообщейний для HTTP сообщений. Подобно [Amazon's HTTP Signature scheme][amazon-http-signature], используется во многих своих сервисах, это разрешение без гражданства, для каждого запроса авторизации. [Elvio Toccalino][etoccalino] поддерживает  пакет [djangorestframework-httpsignature][djangorestframework-httpsignature] для простого использования механизма HTTP подписи. 

## Djoser 

Библиотека [Djoser][djoser] предоставляет набор view для обработки базовых действий таких как регистрация, логирование, выход из системы, сброс пароля и активация аккаунта. Пакет работает с моделями кастомных пользователей и использует авторизацию основаную на токенах. Использует REST авторизационной системы Джанго.

## django-rest-auth

Библиотека [Django-rest-auth][django-rest-auth] предоставляет набор инструментов для REST API: регистрация, авторизация (включая авторизацию через социальные сети), сброс пароля, получение и обновление пользовательских деталей и тд. С этими инструментами, ваше клиентское приложение как AngularJS, iOS, Android и другие смогут взаимодействовать с Джанго бэкендом независимо посредством REST API. 

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
