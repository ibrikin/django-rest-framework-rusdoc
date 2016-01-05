source: permissions.py

# Разрешения

> Аутентификация или идентификация сама по себе не достаточное условие для того чтобы получить доступ к информации или коду. Для этого, тот кто запрашивает доступ должен быть авторизован.
>
> &mdash; [Apple Developer Documentation][cite]

Вместе с [authentication] и [throttling], разрешения определяют следует ли получить или оказать в доступе для запроса

Проверка разрешения всегда запускается в самом начале view, до того как другому коду подет позволено выполнится.  Проверка разрешения будет обычно использована аутентификация информация в `request.user` и `request.auth` для определения разрешения во входящем запросе.

Разрешения используются для разрешения или запрета доступа различных классов пользователей к различным частям API.

Простейший стиль разрешения должен давать разрешение на доступ любому авторизованому пользователю, и отказывать в доступе любому не авторизованому пользователю. Это похоже на `IsAuthenticated` класс в REST framework.

Немного менее строгий стиль разрешения должен позволять полный доступ для авторизованых пользователей, но еще позволять только чтение неавторизованым пользователям. Это связано с `IsAuthenticatedOrReadOnly` классом в REST framework.

## Как разрешения определяются

Разрешение в REST framework постоянно определяются как список класов

Прежде чем запустить тело view каждое разрешение в списке проверяется.
Если какая либо проверка разрешения заканчивается `exceptions.PermissionDenied` или `exceptions.NotAuthenticated`, то главное тело view не будет запущено.

Когда проверка разрешения заканчивается "403 Forbidden" или "401 Unauthorized" ответом, в соответсвии следующим правилам:

* Запрос был аутентифицирован, но в разрешение откразано. *&mdash; Сервер вернет HTTP 403 Forbidden*
* Запрос не аутентифицирован, и приоритетный класс аутентификации *не* использует `WWW-Authenticate` headers. *&mdash; Сервер вернет HTTP 403 Forbidden.*
* Запрос не аутентифицирован, и приоритетный класс аутентификации *таки* использует `WWW-Authenticate` headers. *&mdash; Сервер вернет HTTP 401 Unauthorized, с соответствиющим `WWW-Authenticate` хедером.*

## Object level permissions

Разрешения REST framework также поддерживает объектноуровневое разрешение. Объектно уровневое разрешение определяет следует ли дать разрешение действовать над определенным объектом, который будет обычно моделью экземпляра.

Объектно уровневое разрешение запускается обобщенным REST framework view когда вызывается `.get_object()`.
Так view c уровнем разрешения, получет исключение `exceptions.PermissionDenied` если пользователю не позволено производить действия над объектом.

Если вы написали свое view и хотите соблюсти объектно уровневое разрешение, или вы переопределили метод `get_object`, догда вам надо явно вызвать метод `.check_object_permissions(request, obj)` в месте в котором вы получаете объект.

Это вызовет исключения `PermissionDenied` или `NotAuthenticated` при отсутствии разрешения, или вернет объект при наличие соответсвующего разрешения.

For example:

    def get_object(self):
        obj = get_object_or_404(self.get_queryset())
        self.check_object_permissions(self.request, obj)
        return obj

#### Ограничения объектно уровневых разрешений

По причинам производительности обобщенный view не будет автоматически заполнять объектно уровневые разрешения для каждого экземпляра в запросе, когда возвращает список объектов.

Часто когда вы используете объектно уровневое разрешения, вы также захотите [отфильтровать запросы][filtering] соответсвенно, чтобы убедиться, что пользователи видят только то, что им разрешено.

## Установки разрешающих прав

По умолчанию права разрешений могут быть установлены глобально, используя `DEFAULT_PERMISSION_CLASSES` в файле settings.py. Для примера:

    REST_FRAMEWORK = {
        'DEFAULT_PERMISSION_CLASSES': (
            'rest_framework.permissions.IsAuthenticated',
        )
    }

Если не определены, то по умолчанию стоят не строгие права:

    'DEFAULT_PERMISSION_CLASSES': (
       'rest_framework.permissions.AllowAny',
    )

Вы также можете установить авторизационные права для каждого view, используя классс `APIView`.

    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response
    from rest_framework.views import APIView

    class ExampleView(APIView):
        permission_classes = (IsAuthenticated,)

        def get(self, request, format=None):
            content = {
                'status': 'request was permitted'
            }
            return Response(content)

Или если вы используете декоратор `@api_view`

    from rest_framework.decorators import api_view, permission_classes
    from rest_framework.permissions import IsAuthenticated
    from rest_framework.response import Response

    @api_view('GET')
    @permission_classes((IsAuthenticated, ))
    def example_view(request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)

---

# Справочник API

## AllowAny

Класс `AllowAny` разрешает не строгий доступ, **не важно, был запрос авторизован или нет**.

Это разрешение не обязательно, вы можете добиться таких же результатов используя пустой список или кортеж для настроек разрешений, но вы можете найти это полезным в явном назначении.

## IsAuthenticated

Класс `IsAuthenticated` будет отказывать в доступе любому не авторизованому пользователю, и разрешать в обратной ситуации.

Этот вид разрешения подходит если вы хотите чтобы ваш API  был доступен для зарегестрированных пользователей.

## IsAdminUser

Класс `IsAdminUser`будет отказывать в доступе любому пользователю, кроме тех у которых `user.is_staff` - `True`.

Этот вид разрешения подходит если вы хотите чтобы ваш API был доступен только доверенным администраторам.

## IsAuthenticatedOrReadOnly

Класс `IsAuthenticatedOrReadOnly` будет разрешать любые запросы для авторизованных пользователей. Запросы для не авторизованных пользователей будут только разрешены для методов `GET`, `HEAD`, или `OPTIONS`.

Этот вид разрешения подходит если вы хотите предоставить достпук на чтение для анонимных пользователей и доступ на запись только для авторизованных пользователей.

## DjangoModelPermissions

Этот разрешающий класс связан со стандартным модулем Джанго `django.contrib.auth` [model permissions][contribauth]. Этот тип рахрешения может быть использованя для view которым установлен свойство `.queryset`. Авторизация произойдет только если пользователь *авторизован* и обладает соответсвующим *разрешением*.

* Запрос `POST` требует чтобы пользователь имел `дополнительное` разрешение в моделе.
* `PUT` и `PATCH` требуют чтобы пользователь имел разрешение `change` в моделе.
* Запрос `DELETE` требует чтобы пользователь имел разрешение `delete` в моделе.

Поведение может быть переопределенно чтобы поддерживать собственную модель разрешения. Для примера, вы может захотите включить `view` модель разрешения для `GET` запроса.

Чтобы использовать свою модель разрешений, переопределите `DjangoModelPermissions` и установите свойство `.perms_map`. Обратитесь к исходному коду для деталей.

#### Использование с view которые не включают атрибут `queryset`.

Если вы использует такое разрешениу с view которое использует переопределенный метод `get_queryset()` может не быть атрибута `queryset`. В этом случае мы предлагаем также отмечать это view с queryset, таким образом класс может определить требуемое разрешение. Для примера:

    queryset = User.objects.none()  # Required for DjangoModelPermissions

## DjangoModelPermissionsOrAnonReadOnly

Подобно `DjangoModelPermissions`, но позволяет неавторизованным пользователям иметь доступ только к чтению API.

## DjangoObjectPermissions

Этот класс связан со стандартным Джанго [разрешением][objectpermissions] который позволяет пообъектное разрешение в моделе. В порядке использования этого класса, нам нужно добавить поддержку объектноориентированного разрешения, как [django-guardian][guardian].

Как и с `DjangoModelPermissions`, это разрешение может быть применено только для view у которых есть свойство `.queryset` или метод `.get_quryset()`. Авторизация будет возможна только если пользователь *авторизован* и имеет *необходимое разрешение* и подписан *relevant model permissions*.

* Запрос `POST`  требует пользователя у которого есть разрешение `add` для экземпляра модели.
* `PUT` и `PATCH` требует пользователя к которого есть разрешение `change` для экземпляра модели.
* `DELETE` требует пользователя у которого есть разрешение `delete` для экземпляра модели.

Имейте ввиду что `DjangoObjectPermissions` **не** требует пакета `django-guardian` и также поддерживать и другие бэкенды объектного уровня.

Как и в `DjangoModelPermissions` вы можете использовать свою модель разрешений переопределив `DjangoModelPermissions` и  установив свойство `.perms_map`. Обратитель к исходному коду для деталей.

---

**Примечание**: Если вам нужен объектный уровень разрешений во `view` для `GET`, `HEAD` и `OPRIONS` запросов, вы захотите рассмотреть добавление класса `DjangoObjectPermissionsFilter` для того чтобы убедиться что список конечных точек возвращает результаты включающие объектры для которых пользователь имеет соответсвующее разрешение.

---

---

# Кастомные разрешения

Чтобы применить кастомное разрешение, необходимо переопределить `BasePermission` и применить один из или оба метода:

* `.has_permission(self, request, view)`
* `.has_object_permission(self, request, view, obj)`

Методы должны возвращать `True` если запросы получил доступ, и `False` в другом случае.

Если есть необходимость протестировать методы, это опреации чтения или записи, вам следует проверить методы запроса через константу `SAFE_METHODS` которая является кортежем содержащим `GET`, `OPTIONS` , `HEAD`. Для примера:

    if request.method in permissions.SAFE_METHODS:
        # Check permissions for read-only request
    else:
        # Check permissions for write request

---

**Примечание**: Метод уровня экземпляра `has_object_permission` будет только вызван если на уровне view `has_permission` проверка прошла. Также имейте ввиду, что для запуска проверки на уровне экземпляра, код view должен явно вызвать `.check_object_permissions(request, obj)`.Если вы используете generic view, то это будет сделано по умолчанию.

---

Кастомные разрешения будут показывать исключение `PermissionDenied` если тест провалился. Чтобы изменить сообщение ошибки связанное с исключением, используйте атрибут `message` прямо в вашем кастомном разрешение. В обратном случае атрибут `default_detail` из `PermissionDenied` будет использован.
    
    from rest_framework import permissions

    class CustomerAccessPermission(permissions.BasePermission):
        message = 'Adding customers not allowed.'
        
        def has_permission(self, request, view):
             ...
        
## Примеры

Следующий пример разрешающего класса который проверяет входящие IP адреса запросов с черным списком, и отказывает запросам если они есть в черном списке.

    from rest_framework import permissions

    class BlacklistPermission(permissions.BasePermission):
        """
        Global permission check for blacklisted IPs.
        """

        def has_permission(self, request, view):
            ip_addr = request.META['REMOTE_ADDR']
            blacklisted = Blacklist.objects.filter(ip_addr=ip_addr).exists()
            return not blacklisted

Также как и глобальные разрешения, они проверяют все входящие запросы, вы можете также создать разрешение уровня объекта, которые проверяют операции которые влияют на определенный экземпляр. Для примера:

    class IsOwnerOrReadOnly(permissions.BasePermission):
        """
        Object-level permission to only allow owners of an object to edit it.
        Assumes the model instance has an `owner` attribute.
        """

        def has_object_permission(self, request, view, obj):
            # Read permissions are allowed to any request,
            # so we'll always allow GET, HEAD or OPTIONS requests.
            if request.method in permissions.SAFE_METHODS:
                return True

            # Instance must have an attribute named `owner`.
            return obj.owner == request.user

Заметьте, что generic view будет проверять соответствующее разрешение уровня объекта, но если вы пишите свое кастомное view, вам необходимо будет убедится что разрешение уровня объекта проверяет себя тоже. Вы можете сделать так вызвав `self.check_object_permissions(request, obj)` из view когда получите экземпляр объекта. Этот вызов появится в соответсвии с `APIException` если любое разрешение урованя объекта провалится и в обратном случает просто вернется.

Также заметьте, что generic view будет только проверять разрешение уровня объекта для views полученых единственным экземпляром. Если вам нужно фильтровать список views на уровне объекта, вам необходимо отфильтровать queryset отдельно. Смотри [документацию по фильтрованию][filtering] для подроюностей.

---

# Пакерты третих сторон

Слудующие пакеты доступны.

## Составное разрешение

Пакет [составного разрешения][composed-permissions] предоставляет простой способ определить сложные и глубокие (с логическими операторами) объекты разрешения, используя небольшие и многоразовые компоненты.

## REST условия

Пакет [REST Condition][rest-condition] - это другое расширение для построения сложных разрешений в простой и удобной форме. Расширение позволяет вам комбинировать разрешения с логическими операторами.

## DRY Rest Permissions

Пакет [DRY Rest Permissions][dry-rest-permissions] предоставляет возможность определять различные разрешения для действий по-умолчанию и переопределенных действий. Этот пакет создан для приложений с правами которые получены от отношений определенных в моделе данных приложения. Он также поддерживает проверки разрешений которые возвращаются в приложение клиента через API сериалайзер. Дополнительно он поддерживает добавление разрешений к списку действий чтобы ограничить данные извлекаемые из каждого пользователя.

[cite]: https://developer.apple.com/library/mac/#documentation/security/Conceptual/AuthenticationAndAuthorizationGuide/Authorization/Authorization.html
[authentication]: authentication.md
[throttling]: throttling.md
[filtering]: filtering.md
[contribauth]: https://docs.djangoproject.com/en/dev/topics/auth/customizing/#custom-permissions
[objectpermissions]: https://docs.djangoproject.com/en/dev/topics/auth/customizing/#handling-object-permissions
[guardian]: https://github.com/lukaszb/django-guardian
[get_objects_for_user]: http://pythonhosted.org/django-guardian/api/guardian.shortcuts.html#get-objects-for-user
[2.2-announcement]: ../topics/2.2-announcement.md
[filtering]: filtering.md
[drf-any-permissions]: https://github.com/kevin-brown/drf-any-permissions
[composed-permissions]: https://github.com/niwibe/djangorestframework-composed-permissions
[rest-condition]: https://github.com/caxap/rest_condition
[dry-rest-permissions]: https://github.com/Helioscene/dry-rest-permissions
