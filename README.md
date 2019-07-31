# ImbaChat-docs

## Настройка плагина на нашем сайте

1. Зарегистирируйтесь на сайте http://dev2.imbachat.com.
2. Зайдите на страницу с виджетами ( http://dev2.imbachat.com/admin/widgets ). Нажмите "создать" и заполните поля:
	 - "Секретный ключ" - секретный ключ для формирования JWT токенa.
	 - "Out password" - логин и пароль для аутентификации разрабочтика. Логин и пароль нужно вводить через двоеточие.
	 - "Debug" - включает режим отладки (Не рекомендуется включать).
	 - "URL для получения пользователей" - этот URL должен вызывавать функцию, которую вы напишите на своей серверной части.
	 - "Символ разделения" - разделяет строку с id пользователей, в функции получения пользователей
	 - "URL для авторизации пользователя через логин и пароль"
	 - "Имя"
	 - "no_password" - отключает проверку авторизации (Не рекомендуется включать).
3. Нажмиет "Создать".


## Frontend

1. Для начала нужно подключить javascript ImbaChat'а. Подключение выглядит так `<script src="http://dev2.imbachat.com/imbachat/v1/``DEV_ID``/widget"></script>`
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


## Backend

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
