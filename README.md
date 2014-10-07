## Установка.
[![Gitter](https://badges.gitter.im/Join Chat.svg)](https://gitter.im/berpcor/RBAC?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)

> Перед установкой RBAC необходимо, чтобы была реализована система авторизации/регистрации (средствами Laravel) и имелась таблица Users (в package используется User - модель из коробки).

1. Установить package через composer

```composer
"berpcor/rbac": "dev-master"
```

2. Запустить миграции. 
```shell
  php artisan migrate --package="berpcor/rbac"
```
3. Добавить в config/app.php сервис-провайдер
```php
  'Berpcor\RBAC\RBACServiceProvider',
```
4. Добавить в модель User
```php
  public function role()
  {
      return $this->belongsTo('Role');
  }
```
5. Добавить в фильтры

```php
  Route::filter('rbac', function()
  {
      if(!RBAC::filterMethod()){
          if (Request::ajax())
          {
              $data = array('error'=>'Пользователь не имеет разрешения на доступ.');
              return Response::json($data);
          }else {
              // Или другое требуемое действие в случае, если пользователь не имеет разрешения (редирект, 404, ...)
              App::abort(404); 
          }
      }
  });
```

## Принцип работы

Существует перечень защищенных действий (далее пермишены). Они хранятся в таблице Permissions. Эти пермишены группируются в роли. А роли назначаются пользователю. Т.е. если роли пользователя назначены определенные пермишены, то это значит, что эти действия пользователь может совершать. Те, которые присутствуют в таблице Permissions, но не назначены роли пользователя, пользователь не может совершать.

При создании пользователя, тем же способом, что и до установки rbac, ему по умолчанию назначается роль default.
Роль default запрещает любые защищенные действия. Ее нельзя редактировать и удялять.

Пользователи могут свободно создаваться и удаляться. Роли существуют независимо от пользователей.

Контроль доступа осуществляется на уровне экшенов контроллеров. Они хранятся в таблице Permissions в поле 'action' в виде ControllerName@action. Сохраняемый там экшн берется из маршрута. Если маршрут такой:

```php
Route::get('/test', array(
    "as" => "route_name",
    'before' => 'rbac',
    "uses" => "ControllerName@test"
));
```
То экшн - ControllerName@test. Для того, чтобы контроль учета пользователей работал, необходимо использование маршрутов с контроллерами и с использованием 'uses'. Для такого маршрута контроль работать не будет:

```php
Route::get('/test','ControllerName@action');
```

Далее создается роль. Сразу после создания новая роль не имеет никаких пермишенов и отличается от default только названием и тем, что ей, в отличие от default, можно добавлять пермишены. Т.е. если пользователю назначить созданную роль сразу после ее создания, то он не сможет выполнять ни одного защищенного действия. Для того, чтобы пользователь смог выполнять защищенные действия, его роли нужно добавить пермишены.

Пермишен нельзя удалить, если он назначен хотя бы одной роли. Сначала нужно отвязать у всех ролей этот пермишен. Роль нельзя удалить до тех пор, пока она назначена хотя бы одному пользователю. Чтобы удалить роль, нужно отвязать эту роль у всех пользователей.

Система ролевого доступа используется для пользователей - очевидно. Поэтому, использовать контроль доступа имеет смысл только для авторизованных пользователей. Для того, чтобы система ролевого доступа начала работать, нужно объединить маршруты, для которых необходима авторизация, в группу и первым фильтром назначить auth, а следующим rbac. Проверка того, имеет ли право доступа пользователь к текущему маршруту или нет будет происходить автоматически. Ничего специально делать не нужно. Все работает автоматически. В случае, если для доступа к текущему маршруту используется ajax, то будет возвращаться массив с даными:

```php
$data = array('error'=>'Пользователь не имеет разрешения на доступ.');
return Response::json($data);
```
Поэтому возможно использовать rbac для ajax'а.
## Описание методов
```php

/**
 * Создать роль с именем и ее описанием. Имя роли уникально. При попытке создания роли с дублирующимся именем, выводится  * ошибка. Имя роли вводится на русском. Это имя, вместе с описанием, нужно выводить в админ. разделе сайта.
 */
RBAC::createRole($name, $description);

/**
 * Удалить роль. Удаление возможно только если роль не назначена ни одному пользователю.
 */
RBAC::deleteRole($id);

/**
 * Создание защищенного действия - пермишена. Имя и описание - для человека. Action - для компьютера.
 * Action берется из 'uses' маршрута.
 */
RBAC::createPermission($name, $description, $action);

/**
 * Удаление защищенного действия. Возможно только если оно не назначено ни одной роли.
 */
RBAC::deletePermission($id);

/**
 * Назначание роли пользователю. Если пользователю уже была назначена какая-то роль, то происходит ее переназначение.
 */
RBAC::assignRoleToUser($user_id,$role_id);

/**
 * Абсолютно одинаковые методы, но с разными названиями. Удаляют пользователю текущую назначенную роль. И устанавливают
 * ему роль default. У пользователя не может совсем не быть роли. Если у пользователя уже итак имеется роль default,
 * то выводится ошибка.
 */
RBAC::removeUsersRole($user_id);
RBAC::setDefaultRoleFor($user_id);

/**
 * Роли присваиваются пермишены. в $permission_id должен передаваться массив с id пермишенов. При каждом новом
 * назначении пермишенов старые перезаписываются. Если нужно отвязать все пермишены от роли, то нужно передать пустой 
 * массив id пермишенов.
 */
RBAC::attachPermissionToRole($role_id, $permission_id);

/**
 * Используется для формирования шаблонов. Если требуется выводить или не вывозить что-то в зависимости от того, 
 * разрашено ли пользователю получать доступ к экшену или нет. Испольуется так:
 * if(RBAC::hasPermission('Controller@action')){// разрешено}.
 * В $action передается экшен из uses маршрута.
 */
RBAC::hasPermission($action)
```

Все методы, крому hasPermission могут возвращать ошибку, поэтому их нужно оборачивать в блок try/catch:

```php
    try {
        RBAC::someMethod();
    } catch (Exception $e) {
        // Или любое другое действие
        if (Request::ajax())
        {
            $data = array('error'=>$e->getMessage());
            return Response::json($data);
        }else {
            // Или другое требуемое действие в случае, если пользователь не имеет разрешения (редирект, 404, ...)
            return 'пользователь не имеет разрешения';
            return $e->getMessage();
        }
    }
```

Как и сказано выше, для шаблонов нужно использовать RBAC::hasPermission('action'). Возвращает ```true```, если пользователь имеет право доступа и ```false```, если не имеет.

RBAC работает и RESTful-контроллерами. 

Нужный resource-контроллер прячется за авторизацию (можно добавить в группу со всеми остальными). Или, если он не находится в группе, можно задать фильтры внутри самого контроллера в __construct. Сначала auth, потом - rbac.

## Стандартные роли.

Суперпользователь и Станадартная.
Стандартная роль - все защищенные действия запрещены.
Администратор - все защищенные действия разрешены.
Их нельзя удалять и нельзя редактировать их разрешения. В коде и логике везде идет привязка к их ID в таблице БД. 
При верной установке RBAC-расширения в таблице Roles Стандартная роль должна быть под ID = 1, а Администратор - под ID = 2.
