# Быстрое начало

Мы собираемся создать простое API которое позволит администраторам просматривать и редактировать пользователей и группы пользователей в системе.

## Настройки проекта

Создайте новый Django проект `tutorial`, затем создайте новое приложение `quickstart`.

    # Create the project directory
    mkdir tutorial
    cd tutorial

    # Create a virtualenv to isolate our package dependencies locally
    virtualenv env
    source env/bin/activate  # On Windows use `env\Scripts\activate`

    # Install Django and Django REST framework into the virtualenv
    pip install django
    pip install djangorestframework

    # Set up a new project with a single application
    django-admin.py startproject tutorial .  # Note the trailing '.' character
    cd tutorial
    django-admin.py startapp quickstart
    cd ..

Теперь синхронизируйте вашу базу данных:

    python manage.py migrate

Мы также создадим пользователя `admin` с паролем `password`. Мы будем авторизоваться под этим пользователем позднее в нашем примере.

    python manage.py createsuperuser

После того, как вы настроили базу данных и создали пользователя вы готовы продолжить, откроем папку с приложением и начнем...

## Сериализаторы

Сначала мы определим сериализаторы. Давайте создадим новый модуль `tutorial/quickstart/serializers.py` который мы будем использовать для представления данных.

    from django.contrib.auth.models import User, Group
    from rest_framework import serializers


    class UserSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = User
            fields = ('url', 'username', 'email', 'groups')


    class GroupSerializer(serializers.HyperlinkedModelSerializer):
        class Meta:
            model = Group
            fields = ('url', 'name')

Заметьте, что мы используем гиперлинковое отношение в данном случае, с `HyperlinkedModelSerializer` . Вы также можете использовать первичный ключ и множество других отношений, но гиперлинк хорошо подходит для RESTful.

## Views

Теперь напишем немного во view. Откроем `tutorial/quickstart/views.py` и напечатаем:

    from django.contrib.auth.models import User, Group
    from rest_framework import viewsets
    from tutorial.quickstart.serializers import UserSerializer, GroupSerializer


    class UserViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows users to be viewed or edited.
        """
        queryset = User.objects.all().order_by('-date_joined')
        serializer_class = UserSerializer


    class GroupViewSet(viewsets.ModelViewSet):
        """
        API endpoint that allows groups to be viewed or edited.
        """
        queryset = Group.objects.all()
        serializer_class = GroupSerializer

Вместо того чтобы писать несколько views, мы сгруппировали общие поведения в класс `ViewSets`.

Мы можем просто разделить его на отдельные views если есть необходимость, но использование класса ViewSets позволяет сократить код и организовать логику.

## URLs

Хорошо, теперь давайте подключим к API урлы в `tutorial/urls.py`...

    from django.conf.urls import url, include
    from rest_framework import routers
    from tutorial.quickstart import views

    router = routers.DefaultRouter()
    router.register(r'users', views.UserViewSet)
    router.register(r'groups', views.GroupViewSet)

    # Wire up our API using automatic URL routing.
    # Additionally, we include login URLs for the browsable API.
    urlpatterns = [
        url(r'^', include(router.urls)),
        url(r'^api-auth/', include('rest_framework.urls', namespace='rest_framework'))
    ]

Так как мы используем класс ViewSet вместо View, мы можем автоматически сгенерировать URLconf для нашего API, просто зарегистрировав ViewSets в модуле routers.

Опять, если мы нуждаемся в большем контроле над API URLs мы можем просто использовать обычный класс View и прописать URLconf явно.

В итоге, мы включим стандартный View для логирование и выхода чтобы использовать в браузере. Это опционально, но полезно если ваше API требует авторизации, и вы хотите использовать браузер для просмотра.

## Параметры настройки

Мы также хотели бы установить несколько глобальных свойств. Мы бы включили возможность переключения по страницам, и мы хотели бы чтобы наше API было доступно только админу. Для этого отредактируем `tutorial/settings.py`

    INSTALLED_APPS = (
        ...
        'rest_framework',
    )

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': ('rest_framework.permissions.IsAdminUser',),
        'PAGE_SIZE': 10
    }

Хорошо, API готово.

---

## Тестирование нашего API

Теперь мы готовы к тестированию нашего API. Давайте запустим сервер из командной строки.

    python ./manage.py runserver

Мы можем достучаться до нашего API используя командную строку, либо используя `curl`...

    bash: curl -H 'Accept: application/json; indent=4' -u admin:password http://127.0.0.1:8000/users/
    {
        "count": 2,
        "next": null,
        "previous": null,
        "results": [
            {
                "email": "admin@example.com",
                "groups": [],
                "url": "http://127.0.0.1:8000/users/1/",
                "username": "admin"
            },
            {
                "email": "tom@example.com",
                "groups": [                ],
                "url": "http://127.0.0.1:8000/users/2/",
                "username": "tom"
            }
        ]
    }

Либо используя [httpie][httpie]...

    bash: http -a username:password http://127.0.0.1:8000/users/

    HTTP/1.1 200 OK
    ...
    {
        "count": 2,
        "next": null,
        "previous": null,
        "results": [
            {
                "email": "admin@example.com",
                "groups": [],
                "url": "http://localhost:8000/users/1/",
                "username": "paul"
            },
            {
                "email": "tom@example.com",
                "groups": [                ],
                "url": "http://127.0.0.1:8000/users/2/",
                "username": "tom"
            }
        ]
    }


Или просто используя браузер...

![Quick start image][image]

Если вы используете браузер, убедитесь, что вы залогинились.

Это было просто!

Если вы хотите более глубокого понимания работы REST фраймворка следуйте [руководству][tutorial], или начните изучение [API][guide].

[readme-example-api]: ../#example
[image]: ../img/quickstart.png
[tutorial]: 1-serialization.md
[guide]: ../#api-guide
[httpie]: https://github.com/jakubroztocil/httpie#installation
