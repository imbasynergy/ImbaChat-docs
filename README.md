# ImbaChat-docs

Настройка плагина на нашем сайте

- Зарегистирируйтесь на сайте http://dev2.imbachat.com.
- Зайдите на страницу с виджетами ( http://dev2.imbachat.com/admin/widgets ). Нажмите "создать" и заполните поля:
	 - "Секретный ключ" - секретный ключ для формирования JWT токенa.
	 - "Out password" - логин и пароль для аутентификации разрабочтика. Логин и пароль нужно вводить через двоеточие.
	 - "Debug" - включает режим отладки (Не рекомендуется включать).
	 - "URL для получения пользователей" - этот URL должен вызывавать функцию, которую вы напишите на своей серверной части.
	 - "Символ разделения" - разделяет строку с id пользователей, в функции получения пользователей
	 - "URL для авторизации пользователя через логин и пароль"
	 - "Имя"
	 - "no_password" - отключает проверку авторизации (Не рекомендуется включать).
3. Нажмиет "Создать".
