# ImbaChat-docs


# API чата

Интеграция чата требует реализовать апи интерфейс для обмена данными между чатом и подключемым сайтом

Это необходимо для синхронизации информации о пользователях. На пример чтоб в чате отображались актуальные имена и аватарки пользователей сайта.


## Backend API чата

Примером реализации всех необходимых для чата интерфейсов будет служить этот файл https://github.com/imbasynergy/ImbaChat-OctoberCMS/blob/master/controllers/apiChat.php

# Авторизация

Все запросы от imbachat к интегрируемому сайту содержат параметры для авторизации, позволяющие проверить что запрос действительно прислан от imbachat.com

При настройке параметров виджета в личном кабинете есть параметры именуемые `API login` и `API password` они передаются при обращениях с сервера чата к серверу подключённого сайта в заголовке `Authorization`

Пример проверки авторизации на php.
```php
    $login = Settings::get('dev_login');
    $password = Settings::get('dev_password');

    if(!isset($_SERVER['PHP_AUTH_USER'])
            || ($_SERVER['PHP_AUTH_PW']!=$password) 
            || strtolower($_SERVER['PHP_AUTH_USER'])!=$login)
    {
        header('WWW-Authenticate: Basic realm="Backend"');
        header('HTTP/1.0 401 Unauthorized');
        echo json_encode([
                "code" => 401,
                "version" => $this->getApiVersion()['version'],
                "error" => 'Authenticate required!',
                'debug' => ''
            ]);
        die();
    }
```

Как видим в php параметры `API login` и `API password` из настроек виджета в личном кабинете на http://imbachat.com помещены в `$_SERVER['PHP_AUTH_USER']` и `$_SERVER['PHP_AUTH_PW']`


# Получение информации о людях

В личном кабинете в настройках виджета есть параметр `Users info URL` для указания откуда чат должен брать информацию о пользователях.
 
На адрес `Users info URL` чат будет отправлять запрос передавая список идентификаторов пользователей по которым нужна информация.

В ответ ожидается json со структурой:
```json
[
    {
        "user_id" : 6,
        "name" : "Артём",
    },
    {
        "user_id" : 6,
        "name" : "Артём",
    }
]
```

Поля user_id и name являются обязательными. Так же чат может принимать и обрабатывать дополнительные поля. 
Пример более подробного ответа с информацие о пользователях.

```json
[
    {
     "user_id" : 6,
     "avatar_url" : "http://comet-server.ru/doc/CometQL/Star.Comet-Chat/img/avatar0.png",
     "name" : "Артём",
     "profiles" : {
                    "11" : {
                        "user_id" : "P11-A",
                        "avatar_url" : "http://comet-server.ru/doc/CometQL/Star.Comet-Chat/img/avatar0.png",
                        "name" : "P11-Артём",
                    },
                }
    }
]
```

Обратите внимание что в ответ ожидается именно массив с информацией об одном или более пользователях.

### Пример реализации функции для отдачи информации о пользователях

```php
<?php

// credentials for getting users data from server:
$login = "root";   // Лучше конечно вынести в конфигурационный файл. но для простоты примера захардкодили тут.
$password = "abc";
 
if(!isset($_SERVER['PHP_AUTH_USER']) || ($_SERVER['PHP_AUTH_PW']!=$password) || strtolower($_SERVER['PHP_AUTH_USER'])!=$login)
{
    // AUTH FAIL
    header('WWW-Authenticate: Basic realm="Backend"');
    header('HTTP/1.0 401 Unauthorized');
    echo json_encode([
                "code" => 401,
                "version" => $this->getApiVersion()['version'],
                "error" => 'Authenticate required!',
                'debug' => ''
            ]);
    die();
}

// AUTH SUCCESS

// getting array of user ids:
$id_array = preg_replace("/[^0-9,]/", '', $_GET['user_ids']);

// establishing database connection:
$db_connection = mysqli_connect("localhost", "user", "pass","bd", 3306);

// set users table:
$users_table = "ktvs_users";

// set user ID column:
$users_param = "user_id";

// getting users from the database by their ids:
$sql = "SELECT * FROM $users_table WHERE $users_param IN ( $id_array )";
$users_query = mysqli_query($db_connection, $sql);
$num_rows = mysqli_num_rows($users_query);
$result = [];

if($num_rows>0)
{
    while($rows = mysqli_fetch_assoc($users_query))
    {
        $user_id = $rows['user_id'];
        $user_name = $rows['display_name'];
        $login_date = $rows['last_login_date'];
        $badge = $rows['custom10'];
	if(!empty($rows['avatar']))
	{
        	$avatar = "https://www.example.com/contents/avatars/".$rows['avatar'];
	} else {
        	$avatar = false;
	}
        $link = "https://www.example.com/members/".$user_id."/";

        $result[] = array(
		'user_id' => $user_id,
		'name' => $user_name,
		'avatar_url' => $avatar,
		'profile_url' => $link, 
		'lastlogin' => strtotime($login_date),
		'badge' => $badge
	);
    }
}

// JSON object with users data:
echo json_encode($result);
```

### Авторизации пользователя в чате. 

Для авторизации пользователей подключаемого сайта в чате необходимо правильно сгенерировать JWT токен и передать его как параметр user_key в объекте с другими параметрами инициализации чата.

Для генерации jwt токена, который чат примет как валидный, надо иметь секретный ключ (В личном кабинете подписан как `Secret key`). Его можно получить в форме настроек вашего виджета на сайте imbachat.com
 
Пример реализации функции для генерации токена
```php

function getJWT($user_id, $secret_key, $time)
{
	// Create token header as a JSON string
	$header = json_encode(['typ' => 'JWT', 'alg' => 'HS256']);
	$pass = $secret_key;
	$data = array();
	$data['exp'] = (int)date('U')+$time;
	$data['user_id'] = (int)$user_id;

	// Create token payload as a JSON string
	$payload = json_encode($data);

	// Encode Header to Base64Url String
	$base64UrlHeader = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($header));

	// Encode Payload to Base64Url String
	$base64UrlPayload = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($payload));

	// Create Signature Hash
	$signature = hash_hmac('sha256', $base64UrlHeader . "." . $base64UrlPayload, $pass, true);

	// Encode Signature to Base64Url String
	$base64UrlSignature = str_replace(['+', '/', '='], ['-', '_', ''], base64_encode($signature));

	// Create JWT
	return trim($base64UrlHeader . "." . $base64UrlPayload . "." . $base64UrlSignature);
}
```

Пример вызова функции для генерации токена

```
$user_id = 1;
$secret_key = 'lPXBFPqNg3f661JcegBY0N0dPXqUBdHXqj2cHf04PZgLHxT6z55e20ozojvMRvB8';
echo getJWT($user_id, $secret_key, 3600);

```


## Frontend

### Подключение виджета

1. Для начала нужно подключить javascript ImbaChat'а. Подключение выглядит так `<script src="http://imbachat.com/imbachat/v1/``DEV_ID``/widget"></script>`
, где вместо `DEV_ID` id виджета ( смотрите на странице виджета ).

2. Далее мы вставляем скрипт загрузки чата
```javascript
function imbachatWidget(opt){
    if(!window.ImbaChat){
	return setTimeout(imbachatWidget, 50, opt)
    }
    window.ImbaChat.load(opt);
}
imbachatWidget({
	user_id: "<?= $user_id ?>",
	token: "<?= getJWT($user_id, $secret_key, 3600) ?>"
});
```  

### API виджета

Создание диалога между текущим пользователем и пользователем с id=2 (на пример для реализации кнопки "Написать сообщение" в профиле пользователя)

```javascript
imbaChat.openDialog(2)
```  
