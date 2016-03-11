# Руководство 2: Запросы и Ответы

В этой части руководства мы начнем с основ REST фреймворка. Позвольте представить основные строительные блоки фреймворка.

## Объекты Request

REST фрeймворк использует объект `Request` который расширяет обычный `HttpRequest` и предоставляет более гибкий способ парсить запросы. Ключевая функциональность объекта `Request` заключена в атрибуте `request.data`, который похож на `request.POST`, но лучше для работы с Web API.

    request.POST  # Only handles form data.  Only works for 'POST' method.
    request.data  # Handles arbitrary data.  Works for 'POST', 'PUT' and 'PATCH' methods.

## Объекты Response

REST фрeймворк также использует объект `Response`, который похож на `TemplateResponse` принимающий необработанный контент и использующий согласование содержания чтобы определить правильный тип контента для того, чтобы вернуть клиенту.

    return Response(data)  # Renders to content type as requested by the client.

## Коды состояния

Использование множества HTTP кодов состояния во view делает неочевидным чтение этого кода, и легко не заметить ошибку. REST фреймворк предоставляет более явный способ определить каждый статус, например, `HTTP_400_BAD_REQUEST` в модуле `status`. Это хорошая идея использовать такие коды, нежели использовать численные идентификаторы.

## Оборачиваем API views

REST фреймворк предоставляет два способа обернуть API views.

1. Декоратор `@api_view` для работы с view на основе функций.
2. Класс `APIView` для работы с view на основе классов.

Эти обертки предоставляют некоторую функциональность: возможность убедится, что вы получили экземпляр `Request` в вашем view, и добавление контекста в объект `Response` так, что согласование содержимого может быть произведено.

Обертывание также позволяет вернуть при случае ответы `405 Method Not Allowed` и обработать любое `ParseError` исключение которое появляется при доступе к `request.data` с испорченными данными.

## Объеденим все вместе

Хорошо, давайте пойдем дальше и начнем использовать эти новые компоненты чтобы написать несколько views.

У нас нет больше нужды в нашем классе `JSONResponse` во `views.py`, так что удалим его. После этого мы можем начать рефакторить наше view.

    from rest_framework import status
    from rest_framework.decorators import api_view
    from rest_framework.response import Response
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer


    @api_view(['GET', 'POST'])
    def snippet_list(request):
        """
        List all snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return Response(serializer.data)

        elif request.method == 'POST':
            serializer = SnippetSerializer(data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data, status=status.HTTP_201_CREATED)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

Теперь наше view лучше предыдущего примера. Оно немного короче и код похож по структуре если бы мы работали с Forms API. Мы также используем именованные коды состояния, которые создают значения ответов более очевидными.

Это view для отдельного снипета, в модуле `views.py`.

    @api_view(['GET', 'PUT', 'DELETE'])
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a snippet instance.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return Response(status=status.HTTP_404_NOT_FOUND)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return Response(serializer.data)

        elif request.method == 'PUT':
            serializer = SnippetSerializer(snippet, data=request.data)
            if serializer.is_valid():
                serializer.save()
                return Response(serializer.data)
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

        elif request.method == 'DELETE':
            snippet.delete()
            return Response(status=status.HTTP_204_NO_CONTENT)

Должно быть это все очень знакомо, потому что тут нет больших отличий от обычный Django views.

Заметьте, что мы теперь не явно связываем наши запросы или ответы с типом контента. `request.data` может обрабатывать входящие `json` запросы, но также он может обрабатывать и другие форматы. Аналогично мы возвращаем объекты ответа с данными, но позволяем REST фреймворку обработать ответ в нужный тип контента для нас.

## Добавление опционального расширения для урла

Воспользуемся тем, что наши ответы не привязаны к одному типу контента и добавим поддержку расширений. Это позволит нам явно указывать получаемый формат и означает, что наше API будет принимать URL подобного вида [http://example.com/api/items/4/.json][json-url].

Начнем с добавления аргумента `format` к обоим views

    def snippet_list(request, format=None):

и

    def snippet_detail(request, pk, format=None):

Теперь обновим немного файл `urls.py` добавив набор `format_suffix_patterns` к существующим урлам.

    from django.conf.urls import url
    from rest_framework.urlpatterns import format_suffix_patterns
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)$', views.snippet_detail),
    ]

    urlpatterns = format_suffix_patterns(urlpatterns)

Не обязательно добавлять эти дополнительные паттерны, но это дает нам простой и ясный путь к назначению определенных форматов.

## Как это выглядит?

Давайте протестируем наше API в командной строке, как мы делали в [первой части руководства][tut-1]. Все работает очень похоже, но если мы отправил неверный запрос, то получим обработчик ошибок.

Мы можем получить список всех снипетов как и прежде.

    http http://127.0.0.1:8000/snippets/

    HTTP/1.1 200 OK
    ...
    [
      {
        "id": 1,
        "title": "",
        "code": "foo = \"bar\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      },
      {
        "id": 2,
        "title": "",
        "code": "print \"hello, world\"\n",
        "linenos": false,
        "language": "python",
        "style": "friendly"
      }
    ]

Мы можем контролировать формат ответа используя хедер `Accept`:

    http http://127.0.0.1:8000/snippets/ Accept:application/json  # Request JSON
    http http://127.0.0.1:8000/snippets/ Accept:text/html         # Request HTML

Или добавлением расширения:

    http http://127.0.0.1:8000/snippets.json  # JSON suffix
    http http://127.0.0.1:8000/snippets.api   # Browsable API suffix

Подобно, мы можем контролировать формат запроса, используя хедер `Content-Type`.

    # POST using form data
    http --form POST http://127.0.0.1:8000/snippets/ code="print 123"

    {
      "id": 3,
      "title": "",
      "code": "print 123",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

    # POST using JSON
    http --json POST http://127.0.0.1:8000/snippets/ code="print 456"

    {
        "id": 4,
        "title": "",
        "code": "print 456",
        "linenos": false,
        "language": "python",
        "style": "friendly"
    }

Теперь откройте API в браузере, посетив [http://127.0.0.1:8000/snippets/][devserver]

### Browsability 

Так как API выбрало тип контента ответа на основе клиентского запроса, то по умолчанию, вернет HTML отфарматированный ресурс когда он будет вызван веб браузером. Это позволяет API возвращать полностью веб ориентированное представление.

Обладание вебориентированным API это большая пользовательская победа, которая делает разработку и использование API на много проще. Это также низкий барьер вхождения для других разработчиков которые захотят посмотреть и использовать это API.

Смотри тему [api в браузере][browsable-api] для дополнительной информации о фичах вебориентированного API и как его адаптированить под свои нужды.

## Что дальше?

В [части третьей][tut-3] мы начнем использовать views на основе классов и увидим как общие view сокращают количество кода.

[json-url]: http://example.com/api/items/4/.json
[devserver]: http://127.0.0.1:8000/snippets/
[browsable-api]: ../topics/browsable-api.md
[tut-1]: 1-serialization.md
[tut-3]: 3-class-based-views.md
