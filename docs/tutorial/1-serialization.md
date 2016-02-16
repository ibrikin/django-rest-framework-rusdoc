# Руководство 1: Сериализация

## Введение

Это руководство покрывает процесс создания простого Pastbin сервиса с акцентом на Web API. Заодно познакомит с различными компонентами которые составляют REST framework, и даст полное представление его работы.

Это руководство довольно углубленное, так что вам следует взять печеньки и чашечку чего-нибудь горячего прежде чем начать. Если вы хотите краткий обзор, то следует перейти к [краткой][quickstart] документации.

---

**Примечание**: Код для этого руководства доступен в [tomchristie/rest-framework-tutorial][repo] репозитарии на GitHub. Завершенная реализация также доступна как "песочница" для тестирования [тут][sandbox].

---

## Настраиваем новое окружение

Прежде чем что-то делать, мы создадим новое виртуальное окружение используя [virtualenv]. Это обеспечит хорошую изоляцию для нашей конфигурации модулей от любых других проектов над которыми мы работаем.

    virtualenv env
    source env/bin/activate

Теперь когда мы внутри виртуального окружения, можно установить необходимые нам пакеты.

    pip install django
    pip install djangorestframework
    pip install pygments  # We'll be using this for the code highlighting

**Примечание:** Для того чтобы выйти из виртуального окружения, просто наберите `deactivate`.  Подробную информацию смотрите в [документации virtualenv][virtualenv].

## Начиная

Хорошо, мы готовы кодить.
Чтобы начать, давайте создадим новый проект для работы.

    cd ~
    django-admin.py startproject tutorial
    cd tutorial

Теперь мы можем создать приложение, которое будем использовать для создания простого Web API.

    python manage.py startapp snippets

Нужно добавить наши новые приложения `snippets` и `rest_framework` в `INSTALLED_APPS`. Для этого отредактируем файл `tutorial/settings.py`:

    INSTALLED_APPS = (
        ...
        'rest_framework',
        'snippets',
    )

Также необходимо подключить корневой urlconf в `tutorial/urls.py`, чтобы включить урл нашего приложения.

    urlpatterns = [
        url(r'^', include('snippets.urls')),
    ]

Хорошо, мы готовы продолжить.

## Создание модели

Для этого руководства мы создадим простую модель `Snippet` которая будет сохранять наши снипеты. Для этого отредактируем файл `snippets/models.py`. Примечание: Хорошая практика программирования включает комментарии. Хотя вы и встретите их в нашем репозитарии, здесь мы их опустили для концентрации на коде.

    from django.db import models
    from pygments.lexers import get_all_lexers
    from pygments.styles import get_all_styles

    LEXERS = [item for item in get_all_lexers() if item[1]]
    LANGUAGE_CHOICES = sorted([(item[1][0], item[0]) for item in LEXERS])
    STYLE_CHOICES = sorted((item, item) for item in get_all_styles())


    class Snippet(models.Model):
        created = models.DateTimeField(auto_now_add=True)
        title = models.CharField(max_length=100, blank=True, default='')
        code = models.TextField()
        linenos = models.BooleanField(default=False)
        language = models.CharField(choices=LANGUAGE_CHOICES, default='python', max_length=100)
        style = models.CharField(choices=STYLE_CHOICES, default='friendly', max_length=100)

        class Meta:
            ordering = ('created',)

Также необходимо инициализировать миграцию для нашей модели и синхронизировать базу данных для первого раза.

    python manage.py makemigrations snippets
    python manage.py migrate

## Создание класса Serializer

Первая вещь которую необходимо сделать для начала в нашем Web API - это предоставить способ сериализации и десериализации экземпляров снипетов в `json`. Мы можем сделать это путем объявления сериализаторов которые работают подобно формам в Django. Создадим файл `serializers.py` в папке `snippets` и добавим следующее:

    from rest_framework import serializers
    from snippets.models import Snippet, LANGUAGE_CHOICES, STYLE_CHOICES


    class SnippetSerializer(serializers.Serializer):
        pk = serializers.IntegerField(read_only=True)
        title = serializers.CharField(required=False, allow_blank=True, max_length=100)
        code = serializers.CharField(style={'base_template': 'textarea.html'})
        linenos = serializers.BooleanField(required=False)
        language = serializers.ChoiceField(choices=LANGUAGE_CHOICES, default='python')
        style = serializers.ChoiceField(choices=STYLE_CHOICES, default='friendly')

        def create(self, validated_data):
            """
            Create and return a new `Snippet` instance, given the validated data.
            """
            return Snippet.objects.create(**validated_data)

        def update(self, instance, validated_data):
            """
            Update and return an existing `Snippet` instance, given the validated data.
            """
            instance.title = validated_data.get('title', instance.title)
            instance.code = validated_data.get('code', instance.code)
            instance.linenos = validated_data.get('linenos', instance.linenos)
            instance.language = validated_data.get('language', instance.language)
            instance.style = validated_data.get('style', instance.style)
            instance.save()
            return instance

Первая часть класса определяет поля, которые будут сериализировать/десериализировать. Методы `create()` и `update()` определяют как экземпляры будут создаваться или изменяться когда вызывается `serializer.save()`

Класс сериализации очень похож на класс `Form` Django фреймворка и включает похожие валидационные флаги для различных полей, такие как `required`, `max_length` и `default`.

Флаги полей также могут контролировать как сериалайзер будет отображен в определенных обстоятельствах при рендеринге HTML. Флаг `{'base_template': 'textarea.html'}` указанный выше, эквивалентен использованию `widget=widgets.Textarea` в классе `Form`. Как мы увидим дальше в руководстве, очень полезно контролировать как браузеру следует отображать API.

В действительности мы также можем сохранить для себя некоторое время используя класс `ModelSerializer`, как мы увидим позднее, но сейчас мы определили сериалайзер явно.

## Работаем с сериалайзерами

Прежде чем мы пойдем дальше, мы познакомимся поближе с использованием нашего класса Serializer. Давайте перейдем к Django шеллу.

    python manage.py shell

Хорошо, после того как мы сделаем несколько импортом, давайте создадим пару снипетов для работы.

    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser

    snippet = Snippet(code='foo = "bar"\n')
    snippet.save()

    snippet = Snippet(code='print "hello, world"\n')
    snippet.save()

У нас теперь есть снипеты для работы. Давайте посмотрим на сериализацию одного из них.

    serializer = SnippetSerializer(snippet)
    serializer.data
    # {'pk': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}

В этом месте мы переводим экземпляр модели в нативные типы данных Питона. Чтобы закончить процесс сериализации мы преобразуем данные в `json`.

    content = JSONRenderer().render(serializer.data)
    content
    # '{"pk": 2, "title": "", "code": "print \\"hello, world\\"\\n", "linenos": false, "language": "python", "style": "friendly"}'

Десериализация подобна. Для начала мы парсим поток нативных типов данных Питона...

    from django.utils.six import BytesIO

    stream = BytesIO(content)
    data = JSONParser().parse(stream)

...затем мы востанавливаем эти нативные типы данных в полностью заполненный экземпляр.

    serializer = SnippetSerializer(data=data)
    serializer.is_valid()
    # True
    serializer.validated_data
    # OrderedDict([('title', ''), ('code', 'print "hello, world"\n'), ('linenos', False), ('language', 'python'), ('style', 'friendly')])
    serializer.save()
    # <Snippet: Snippet object>

Заметно как похоже это API на работу с формами. Сходство станет более заметным, когда мы начнем писать views используя наш сериалайзер.

Мы может также сериализовать запросы вместо экземпляра модели. Чтобы сделать так, просто добавьте флаг `many=True` в аргументы сериалайзера. 

    serializer = SnippetSerializer(Snippet.objects.all(), many=True)
    serializer.data
    # [{'pk': 1, 'title': u'', 'code': u'foo = "bar"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}, {'pk': 2, 'title': u'', 'code': u'print "hello, world"\n', 'linenos': False, 'language': u'python', 'style': u'friendly'}]

## Использование ModelSerializers

Класс `SnippetSerializer` дублирует много информации, которая уже содержится в модели `Snippet`. Было бы здорово сократить наш код.

По этой же причине Django содержит класс `Form` и класс `ModelForm`, так и REST фреймворк включает в себя класс `Serializer` и класс `ModelSerializer`.

Давайте посмотрим на рефакторинг нашего сериализатора используя класс `ModelSerializer`. 
Откроем снова файл `snippets/serializers.py` и заменим класс `SnippetSerializer` следующим.

    class SnippetSerializer(serializers.ModelSerializer):
        class Meta:
            model = Snippet
            fields = ('id', 'title', 'code', 'linenos', 'language', 'style')

У сериализатора есть замечательное свойство, которое позволяет просмотреть все поля в экземпляре напечатав его представление. Откройте Django шелл `python manage.py shell` и попробуйте следующее:

    >>> from snippets.serializers import SnippetSerializer
    >>> serializer = SnippetSerializer()
    >>> print(repr(serializer))
    SnippetSerializer():
        id = IntegerField(label='ID', read_only=True)
        title = CharField(allow_blank=True, max_length=100, required=False)
        code = CharField(style={'base_template': 'textarea.html'})
        linenos = BooleanField(required=False)
        language = ChoiceField(choices=[('Clipper', 'FoxPro'), ('Cucumber', 'Gherkin'), ('RobotFramework', 'RobotFramework'), ('abap', 'ABAP'), ('ada', 'Ada')...
        style = ChoiceField(choices=[('autumn', 'autumn'), ('borland', 'borland'), ('bw', 'bw'), ('colorful', 'colorful')...

Важно помнить, что класс `ModelSerializer` не делает ничего магического, это просто сокращенный способ создания сериализатора:

* Автоматически определяет набор полей.
* Простая реализация методов `create()` и `update()` по умолчанию.

## Создание обычного Django view используя наш сериализатор

Давайте посмотрим, как мы можем написать API view используя наш новый класс.
Сейчас мы не будем использовать инструменты REST фреймворка, а просто создадим обычное Django view.

Начнем с создания подкласса HttpResponse который мы будем использовать для преобразования любых данных в `json`.

Отредактируйте файл `snippets/views.py` и добавьте следующее.

    from django.http import HttpResponse
    from django.views.decorators.csrf import csrf_exempt
    from rest_framework.renderers import JSONRenderer
    from rest_framework.parsers import JSONParser
    from snippets.models import Snippet
    from snippets.serializers import SnippetSerializer

    class JSONResponse(HttpResponse):
        """
        An HttpResponse that renders its content into JSON.
        """
        def __init__(self, data, **kwargs):
            content = JSONRenderer().render(data)
            kwargs['content_type'] = 'application/json'
            super(JSONResponse, self).__init__(content, **kwargs)

Корень нашего API будет поддерживать вывод списка всех существующий снипетов или создание новых.

    @csrf_exempt
    def snippet_list(request):
        """
        List all code snippets, or create a new snippet.
        """
        if request.method == 'GET':
            snippets = Snippet.objects.all()
            serializer = SnippetSerializer(snippets, many=True)
            return JSONResponse(serializer.data)

        elif request.method == 'POST':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data, status=201)
            return JSONResponse(serializer.errors, status=400)

Замете что, если мы хотим иметь возможность отправлять POST запросы к нашему view от клиента у которого нет CSRF токена, нам надо использовать `csrf_exempt`. Это не то, что бы вы хотели обычно сделать, и REST фреймворк использует более разумный способ для этого, и этот способ подойдет для наших целей.

Нам также понадобится view которое связанно с отдельным снипетом, и может получать, обновлять и удалять снипеты.

    @csrf_exempt
    def snippet_detail(request, pk):
        """
        Retrieve, update or delete a code snippet.
        """
        try:
            snippet = Snippet.objects.get(pk=pk)
        except Snippet.DoesNotExist:
            return HttpResponse(status=404)

        if request.method == 'GET':
            serializer = SnippetSerializer(snippet)
            return JSONResponse(serializer.data)

        elif request.method == 'PUT':
            data = JSONParser().parse(request)
            serializer = SnippetSerializer(snippet, data=data)
            if serializer.is_valid():
                serializer.save()
                return JSONResponse(serializer.data)
            return JSONResponse(serializer.errors, status=400)

        elif request.method == 'DELETE':
            snippet.delete()
            return HttpResponse(status=204)

В конце нам нужно подключить эти views. Создадим файл `snippets/urls.py`:

    from django.conf.urls import url
    from snippets import views

    urlpatterns = [
        url(r'^snippets/$', views.snippet_list),
        url(r'^snippets/(?P<pk>[0-9]+)/$', views.snippet_detail),
    ]

Стоит отметить, что есть несколько граничных случаев с которыми мы не разобрались должным образом. Если мы отправим неправильно сформированный `json`, или если запрос будет совершен методом, который не обрабатывает view, будет ответ с ошибкой 500. 


## Тестирование нашего первого Web API

Теперь мы можем запустить образцовый сервер для наших снипетов.

Выйдем из шелла...

	quit()

...и запустим Django сервер.

	python manage.py runserver

	Validating models...

	0 errors found
	Django version 1.8.3, using settings 'tutorial.settings'
	Development server is running at http://127.0.0.1:8000/
	Quit the server with CONTROL-C.

В другом окне терминала, мы сможем тестировать сервер.

Мы можем тестировать наше API используя [curl][curl] или [httpie][httpie]. Httpie - это дружелюбный http клиент написанный на Python. Давайте установим его.

Вы можете установить httpie используя pip:

    pip install httpie

В итоге мы можем получить список всех снипетов:

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

Или мы можем получить определенный снипет по его id: 

    http http://127.0.0.1:8000/snippets/2/

    HTTP/1.1 200 OK
    ...
    {
      "id": 2,
      "title": "",
      "code": "print \"hello, world\"\n",
      "linenos": false,
      "language": "python",
      "style": "friendly"
    }

По аналогии, вы можете получить эти же json через браузер посетив те же URLs.

## Где мы сейчас

У нас все хорошо получилось, мы создали API для сериализации которое очень похоже на Django Form API, и несколько обычных Django views.

Наши API views не делают ничего особенного в данный момент, кроме обслуживания `json` ответов и некоторой обработки ошибок, связанных с крайними значениями, но это вполне работающее Web API.

Во [второй части руководства][tut-2] мы увидим как можно улучшить этот код.

[quickstart]: quickstart.md
[repo]: https://github.com/tomchristie/rest-framework-tutorial
[sandbox]: http://restframework.herokuapp.com/
[virtualenv]: http://www.virtualenv.org/en/latest/index.html
[tut-2]: 2-requests-and-responses.md
[httpie]: https://github.com/jakubroztocil/httpie#installation
[curl]: http://curl.haxx.se
