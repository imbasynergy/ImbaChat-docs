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
``` 
    $login = Settings::get('dev_login');
    $password = Settings::get('dev_password');

    if(!isset($_SERVER['PHP_AUTH_USER'])
            || ($_SERVER['PHP_AUTH_PW']!=$password) 
            || strtolower($_SERVER['PHP_AUTH_USER'])!=$login)
    {
        header('WWW-Authenticate: Basic realm="Backend"');
        header('HTTP/1.0 401 Unauthorized');
        echo 'Authenticate required!';
        die();
    }
```

Как видим в php параметры `API login` и `API password` из настроек виджета в личном кабинете на http://imbachat.com помещены в $_SERVER['PHP_AUTH_USER'] и $_SERVER['PHP_AUTH_PW']


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
                        "user_id" : "P11-".$value,
                        "avatar_url" : "http://comet-server.ru/doc/CometQL/Star.Comet-Chat/img/avatar0.png",
                        "name" : "P11-"."Артём".$value,
                    },
                }
    }
]
```

Обратите внимание что в ответ ожидается именно массив с информацией об одном или более пользователях.


## Frontend

1. Для начала нужно подключить javascript ImbaChat'а. Подключение выглядит так `<script src="http://imbachat.com/imbachat/v1/``DEV_ID``/widget"></script>`
, где вместо `DEV_ID` id виджета ( смотрите на странице виджета ).

2. Далее мы вставляем скрипт загрузки чата
```
function imbachatWidget(){
    if(!window.ImbaChat){
	return setTimeout(imbachatWidget, 50)
    }
    window.ImbaChat.load(PARAMETRS);
}
imbachatWidget();
```
Вместо `PARAMETRS` должны быть параметры такого вида:
```
{
	user_id: id текущего пользователя,
	token: JWT токен
}
```

### Функция позволяющая получить информацию о пользователях для ImbaChat.
```
public function getUser($str_ids){
	//Логин и пароль разработчика
	$login = \Config::get('imbasynergy.imbachatwidget::login');
	$password = \Config::get('imbasynergy.imbachatwidget::password');

	//Аутентификация разработчика
	if(!isset($_SERVER['PHP_AUTH_USER']) || ($_SERVER['PHP_AUTH_PW']!=$password) || strtolower($_SERVER['PHP_AUTH_USER'])!=$login)
	{
	    header('WWW-Authenticate: Basic realm="Backend"');
	    header('HTTP/1.0 401 Unauthorized');
	    echo 'Authenticate required!';
	    die();
	}

	//Разделение строки с id пользователей, через символ, который вы прописали в поле "Символ разделения"
	$ids = explode("cимвол разделения", $str_ids);
	$users = array();

	//Заполнение массива с пользователями
	foreach ($ids as $id){
	    $user_m = Получение пользователя по id;
	    $user = array();
	    $user['name'] = Имя пользователя;
	    $user['user_id'] = $id;
	    $user['avatar_url'] = Аватар пользователя ( не обязательно );
	    $users[] = $user;
	}
	return json_encode($users);
}
```

### Функция отдающая JWT токен.
```
function getJWT(){

	// Create token header as a JSON string
	$header = json_encode(['typ' => 'JWT', 'alg' => 'HS256']);
	$pass = Секретный ключ;
	$data = array();
	$data['exp'] = (int)date('U')+3600*7;
	$data['user_id'] = id текущего пользователя;

	if(isset($data['user_id']))
	{
	    $data['user_id'] = (int)$data['user_id'];
	}

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
