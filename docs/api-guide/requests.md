source: request.py

# Requests | Запросы

> If you're doing REST-based web service stuff ... you should ignore request.POST.
> Если вы делаете web REST сервис ... вам следует игнорировать request.POST.
>
> &mdash; Malcom Tredinnick, [Django developers group][cite]

REST framework's `Request` class extends the standard `HttpRequest`, adding support for REST framework's flexible request parsing and request authentication.
Класс `Request` REST фреймворка расширяет стандарт `HttpRequest`, добавляя поддержку для REST парсинга запроса и аутентификацию.

---

# Request parsing | Парсинг запроса

REST framework's Request objects provide flexible request parsing that allows you to treat requests with JSON data or other media types in the same way that you would normally deal with form data.
Request объект предоставляет гибкий парсинг запросов который позволяет вам обращать запросы с JSON или другими типами медия таким же способом как бы вы работали с данными форм.

## .data

`request.data` returns the parsed content of the request body.  This is similar to the standard `request.POST` and `request.FILES` attributes except that:
`request.data` возвращает распарсенные данные тела запроса. Это похоже на стандартные атрибуты `request.POST` и `request.FILES` кроме этого:

* It includes all parsed content, including *file and non-file* inputs.
* Он включает все данные, включая *file и non-file*
* It supports parsing the content of HTTP methods other than `POST`, meaning that you can access the content of `PUT` and `PATCH` requests.
* Он поддерживает обработку данных HTTP методов кроме `POST`, что значит вы можете иметь доступ к данным `PUT` и `PATCH` запросов.
* It supports REST framework's flexible request parsing, rather than just supporting form data.  For example you can handle incoming JSON data in the same way that you handle incoming form data.
* Он поддерживате гибкую обработку запросов вместо простой поддержки данных форм. Для примера вы можете обработать входящий JSON таким же способом как вы обрабатываете входящие данные форм.

For more details see the [parsers documentation]. Подробнее смотри в [документации парсинга][parsers documentation].

## .query_params

`request.query_params` is a more correctly named synonym for `request.GET`.
`request.query_params` - это синоним для `request.GET`.

For clarity inside your code, we recommend using `request.query_params` instead of the Django's standard `request.GET`. Doing so will help keep your codebase more correct and obvious - any HTTP method type may include query parameters, not just `GET` requests.
Для ясности кода мы рекомендуем использовать `request.query_params` вместо стандартного для Джанго `request.GET`. Делая так, поможет вам держать ваш код более правильным и очевидным - любой HTTP тип метода может включать запрос параметров, а не просто `GET` запрос.

## .parsers

The `APIView` class or `@api_view` decorator will ensure that this property is automatically set to a list of `Parser` instances, based on the `parser_classes` set on the view or based on the `DEFAULT_PARSER_CLASSES` setting.
Класс `APIView` или `@api_view` декоратор будет устанавливать это свойство автоматически, устанавливая список экземпляров `Parser` основанных на `parser_classes` установленых во view или основаных на настройках `DEFAULT_PARSER_CLASSES`.

You won't typically need to access this property.
Обычно это свойство вам не нужно.

---

**Note:** If a client sends malformed content, then accessing `request.data` may raise a `ParseError`.  By default REST framework's `APIView` class or `@api_view` decorator will catch the error and return a `400 Bad Request` response.
**Примечание:** Если клиент отправляет неправильно сформированные данные, `request.data` может выкинуть `ParseError` исключение. По-умолчанию класс `APIView` или `@api_view` декоратор будет ловить ошибки и возвращать ответ `400 Bad Request`.

If a client sends a request with a content-type that cannot be parsed then a `UnsupportedMediaType` exception will be raised, which by default will be caught and return a `415 Unsupported Media Type` response.
Если клиент отправляет запрос с content-type которые не может быть распарсен будет вызвано исключение `UnsupportedMediaType`, которое по умолчанию будет поймано и вернет `415 Unsupported Media Type` ответ.

---

# Content negotiation | Соглосования данных

The request exposes some properties that allow you to determine the result of the content negotiation stage. This allows you to implement behaviour such as selecting a different serialisation schemes for different media types.
Запрос раскрывает некоторые свойства которые позволяют вам определить результаты на этапе взаимодействия данных. Это позволяет вам создавать поведение такое как выбор различных сериализационных схем для различных типов медия.

## .accepted_renderer

The renderer instance what was selected by the content negotiation stage.
Экземпляр рендера который был выбран на этапе взаимодействия данных

## .accepted_media_type

A string representing the media type that was accepted by the content negotiation stage.
Строка представляющая тип медия принятый на этапе взаимодействия данных.

---

# Authentication | Аутентификация

REST framework provides flexible, per-request authentication, that gives you the ability to:
REST framework предоставляет гибкую, по запросную аутентификацию, которая дает вам возможность:

* Use different authentication policies for different parts of your API.
* Использовать различные аутентификационные права для различных частей вашего API
* Support the use of multiple authentication policies.
* Поддержку использования множества аутентификационных прав.
* Provide both user and token information associated with the incoming request.
* Предоставляет как пользователям так и токенам информацию связаную с входящими запросами.

## .user

`request.user` typically returns an instance of `django.contrib.auth.models.User`, although the behavior depends on the authentication policy being used.
`request.user` обычно возвращает экземпляр `django.contrib.auth.models.User`, хотя поведение зависит от использованной аутентификационной схемы.

If the request is unauthenticated the default value of `request.user` is an instance of `django.contrib.auth.models.AnonymousUser`.
Если запрос авторизован, по-умолчанию значение `request.user` - это экземпляр `django.contrib.auth.models.AnonymousUser`.

For more details see the [authentication documentation].
Подробности смотри в [документациия аутентификации][authentication documentation].

## .auth

`request.auth` returns any additional authentication context.  The exact behavior of `request.auth` depends on the authentication policy being used, but it may typically be an instance of the token that the request was authenticated against.
`request.auth` возвращает любую дополнительный аутентификационный контекст. Точное поведение `request.auth` зависит от аутентификационных прав которые используются, но это обычно бывает экземпляр токена с которым был авторизован запрос.

If the request is unauthenticated, or if no additional context is present, the default value of `request.auth` is `None`.
Если запрос не авторизован или если нет дополнительных данных, то по-умолчанию значение `request.auth` будет `None`.

For more details see the [authentication documentation].
Подробности смотри в [документациия аутентификации][authentication documentation].

## .authenticators

The `APIView` class or `@api_view` decorator will ensure that this property is automatically set to a list of `Authentication` instances, based on the `authentication_classes` set on the view or based on the `DEFAULT_AUTHENTICATORS` setting.
Класс `APIView` или `@api_view` декоратор будет устанавливать это свойство автоматически, устанавливая список экземпляров `Authentication` основанных на `authentication_classes` установленых во view или основаных на настройках `DEFAULT_PARSER_CLASSES`.

You won't typically need to access this property.
Обычно это свойство вам не нужно.

---

# Browser enhancements | Улучшения

REST framework supports a few browser enhancements such as browser-based `PUT`, `PATCH` and `DELETE` forms.
REST framework поддерживает несколько улучшений такие как формы для `PUT`, `PATCH` и `DELETE`.

## .method

`request.method` returns the **uppercased** string representation of the request's HTTP method.
`request.method` возвращает строку **uppercased** представляющую HTTP метод запроса.

Browser-based `PUT`, `PATCH` and `DELETE` forms are transparently supported.
Браузерные формы `PUT`, `PATCH` и `DELETE` прозрачно поддерживаются.

For more information see the [browser enhancements documentation].
Подробности смотри в [документации к улучшения браузера][browser enhancements documentation].

## .content_type

`request.content_type`, returns a string object representing the media type of the HTTP request's body, or an empty string if no media type was provided.
`request.content_type` возвращает строку представляющую тип тела HTTP запроса, или пустую строку если никакого медия типа не было предоставлено.

You won't typically need to directly access the request's content type, as you'll normally rely on REST framework's default request parsing behavior.
Вам вероятно не понадобится прямой доступ к типу контента запроса, так как вы скорее всего будете пологаться на стандартное поведение запроса у REST Fremework.

If you do need to access the content type of the request you should use the `.content_type` property in preference to using `request.META.get('HTTP_CONTENT_TYPE')`, as it provides transparent support for browser-based non-form content.
Если вам необходим доступ к типу контента запроса, вам следует использовать свойство `.content_type` вместо `request.META.get('HTTP_CONTENT_TYPE')`, так как этот спрособ предлагает прозрачную поддержку контента не из форм на основе браузера.

For more information see the [browser enhancements documentation]. Для подробной информации смотрите [документацию][browser enhancements documentation].

## .stream

`request.stream` returns a stream representing the content of the request body.
`request.stream` возвращает поток представляющий содержание тела запроса.

You won't typically need to directly access the request's content, as you'll normally rely on REST framework's default request parsing behavior.
Вам вероятно не понадобится прямой доступ к типу контента запроса, так как вы скорее всего будете пологаться на стандартное поведение запроса у REST Fremework.

If you do need to access the raw content directly, you should use the `.stream` property in preference to using `request.content`, as it provides transparent support for browser-based non-form content.
Если вам необходим доступ к сырым данным напрямую, вам следует использовать свойство `.stream` вместо `request.content`, так как этот спрособ предлагает прозрачную поддержку контента не из форм на основе браузера.

For more information see the [browser enhancements documentation]. Подробности смотри в [документации][browser enhancements documentation].

---

# Standard HttpRequest attributes

As REST framework's `Request` extends Django's `HttpRequest`, all the other standard attributes and methods are also available.  For example the `request.META` and `request.session` dictionaries are available as normal.
Так как `Request` REST Framework расширяет `HttpRequest` Django, то все другие стандартные атрибуты и методы также доступны. Для примера словари `request.META` и `request.session` доступны как обычно.

Note that due to implementation reasons the `Request` class does not inherit from `HttpRequest` class, but instead extends the class using composition.
Имейте ввиду что в связи с особенностями реализации класса `Request` он не наследуется от класса `HttpRequest`, но используется композиция.


[cite]: https://groups.google.com/d/topic/django-developers/dxI4qVzrBY4/discussion
[parsers documentation]: parsers.md
[authentication documentation]: authentication.md
[browser enhancements documentation]: ../topics/browser-enhancements.md
