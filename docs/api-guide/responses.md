source: response.py

# Responses

> Unlike basic HttpResponse objects, TemplateResponse objects retain the details of the context that was provided by the view to compute the response.  The final output of the response is not computed until it is needed, later in the response process.
>
> В отличие от базовых HttpResponse объектов, TemplateResponse объекты сохраняют детали контекста который был предоставлен view чтобы вычислить ответ. Конечные выводы ответа не вычиляются до необходимости, позднее в процессе ответа.
> &mdash; [Django documentation][cite]

REST framework supports HTTP content negotiation by providing a `Response` class which allows you to return content that can be rendered into multiple content types, depending on the client request.
REST framework поддерживает взаимодействие HTTP контента по средствам класса `Response` который позволяет вам вернуть контент который может быть преобразовано во множество типов, в зависимости от запроса клиента.

The `Response` class subclasses Django's `SimpleTemplateResponse`.  `Response` objects are initialised with data, which should consist of native Python primitives.  REST framework then uses standard HTTP content negotiation to determine how it should render the final response content.
Класс `Response` является подклассом Джанго класса `SimpleTemplateResponse`. Объекты `Response` инициализируются данными, которые должны состоять из примитивных типов данных Python. REST Framework затем использует стандартное взаимодействие HTTP контента чтобы определить как будет преобразован конечный ответ.

There's no requirement for you to use the `Response` class, you can also return regular `HttpResponse` or `StreamingHttpResponse` objects from your views if required.  Using the `Response` class simply provides a nicer interface for returning content-negotiated Web API responses, that can be rendered to multiple formats.
Не обязательно использовать только класс `Response`, вы также можете вернуть обычный `HttpResponse` или `StreamingHttpResponse` объекты из вашего view если это понадобится. Использования класса `Resonse` просто дает лучший интерфейст для возврата содержательного Web API ответа, что затем может быть конвертированно во множество форматов.

Unless you want to heavily customize REST framework for some reason, you should always use an `APIView` class or `@api_view` function for views that return `Response` objects.  Doing so ensures that the view can perform content negotiation and select the appropriate renderer for the response, before it is returned from the view.
До тех пор пока вы не захотите сильно изменить REST framework по каким-то причинам, вам следует всегда использовать класс `APIView` или `@api_view` для view которые возвращают `Response` объекты. Действуя так, убедитесь что view может производить взаимодействие контента и выбирать соответсвующее положение для ответа, до того как его вернет view.
---

# Creating responses | Создание ответов

## Response()

**Signature:** `Response(data, status=None, template_name=None, headers=None, content_type=None)`

Unlike regular `HttpResponse` objects, you do not instantiate `Response` objects with rendered content.  Instead you pass in unrendered data, which may consist of any Python primitives.
В отличие от обычных объектов `HttpResponse`, вам не надо создавать объекты `Response` с преобразованным контентом. Вместо этого, вы отправляете необработанные данные, которые могут содержать любой примитив Python.

The renderers used by the `Response` class cannot natively handle complex datatypes such as Django model instances, so you need to serialize the data into primitive datatypes before creating the `Response` object.
Обработчики используемые классом `Response` не могут обрабатывать сложные типы данных, такие как экземпляры Джанго модели, так что вам необходимо сначала сериализовать данные в примитивы, прежде чем создавать объект `Response`.

You can use REST framework's `Serializer` classes to perform this data serialization, or use your own custom serialization.
Вы можете использовать класс `Serializer` для этой работы, или использовать любой ваш собственный класс.

Arguments: Аргументы:

* `data`: The serialized data for the response.
* `data`: Сериализованные данные для ответа.
* `status`: A status code for the response.  Defaults to 200.  See also [status codes][statuscodes].
* `status`: Статус код для ответа. По умолчанию - 200. Смотрите также [статус коды][statuscodes].
* `template_name`: A template name to use if `HTMLRenderer` is selected.
* `template_name`: Имя шаблона для использования если `HTMLRenderer` выбран.
* `headers`: A dictionary of HTTP headers to use in the response.
* `headers`: Словарь HTTP хедерова для использования в ответе.
* `content_type`: The content type of the response.  Typically, this will be set automatically by the renderer as determined by content negotiation, but there may be some cases where you need to specify the content type explicitly.
* `content_type`: Контент тип ответа. Обычно, он будет установлен автоматически обработчиком как определен взаимодействием данных, но бывают случаи когда есть необходимость определить тип контента явно.

---

# Attributes Атрибуты

## .data

The unrendered content of a `Request` object.
Необрабатываемый контент объекта `Request`

## .status_code

The numeric status code of the HTTP response.
Цифровой статус кода HTTP ответа.

## .content

The rendered content of the response.  The `.render()` method must have been called before `.content` can be accessed.
Обработанный контент ответа. Метод `.render()` должен быть вызван до доступности `.content`.

## .template_name

The `template_name`, if supplied.  Only required if `HTMLRenderer` or some other custom template renderer is the accepted renderer for the response.
`template_name` - если есть. Требуется только если `HTMLRenderer` или какой-либой другой обработчки шаблонов применяемый для ответа.

## .accepted_renderer

The renderer instance that will be used to render the response.
Экземпляр обработчика который будет использоватся для ответа.

Set automatically by the `APIView` or `@api_view` immediately before the response is returned from the view.
Устанавливается автоматически `APIView` или `@api_view` сразу до ответа возращенного из view.

## .accepted_media_type

The media type that was selected by the content negotiation stage.
Тип медиа который был выбран на этапе взаимодействия контента.

Set automatically by the `APIView` or `@api_view` immediately before the response is returned from the view.
Устанавливается автоматически `APIView` или `@api_view` сразу до ответа возращенного из view.

## .renderer_context

A dictionary of additional context information that will be passed to the renderer's `.render()` method.
Словарь дополнительной информации которая будет от

Set automatically by the `APIView` or `@api_view` immediately before the response is returned from the view.
Устанавливается автоматически `APIView` или `@api_view` сразу до ответа возращенного из view.

---

# Standard HttpResponse attributes Стандартные атрибуты HttpResponse

The `Response` class extends `SimpleTemplateResponse`, and all the usual attributes and methods are also available on the response.  For example you can set headers on the response in the standard way:
Класс `Response` расширяет `SimplrTemplateResponse` и все обычные атрибуты и методы которые доступны для ответа. Для примера вы можете установить хедеры для ответа стандартным способом:

    response = Response()
    response['Cache-Control'] = 'no-cache'

## .render()

**Signature:** `.render()`

As with any other `TemplateResponse`, this method is called to render the serialized data of the response into the final response content.  When `.render()` is called, the response content will be set to the result of calling the `.render(data, accepted_media_type, renderer_context)` method on the `accepted_renderer` instance.
Как с любым другим `TemplateResponse`, этот метод вызывается для обработки сериализованных данных ответа в конечный контент ответ. Когда `.render()` вызывается, контент ответа будет устанавливать результат вызова `.render(data, accepted_media_type, renderer_context)` экземпляра `accepted_renderer`.

You won't typically need to call `.render()` yourself, as it's handled by Django's standard response cycle.
Вам обычно не понадобится вызывать `.render()` самому, как это сделано в Джанго.

[cite]: https://docs.djangoproject.com/en/dev/ref/template-response/
[statuscodes]: status-codes.md
