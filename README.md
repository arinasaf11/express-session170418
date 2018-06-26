# I. Создание веб-приложения с аутентификацией на основе сессий
Цель: создать серверное приложение, которое получает массив объектов, каждый из которых имеет свойство login, из адреса https://kodaktor.ru/j/users и использует шаблон pug для вывода в веб-страницу списка этих логинов при условии ввода верных логина и пароля, которые хранятся по указанному адресу.  
Берем за основу прошлое задание (https://github.com/arinasaf11/express200318). Теперь при попытке перейти по маршруту /users должна происходить проверка того, что сессия установлена, и только в этом случае должен быть показан список логинов; иначе должен быть осуществлён переход на форму для ввода логина и пароля.
### 1. Рефакторинг кода в index.js - чтобы содержимое документа со списком логинов и паролей было доступно после инициализации приложения:
```javascript
const express = require('express'); 
const { get } = require('axios'); 

let items; 
const PORT = 4321; 
const URL = 'https://kodaktor.ru/j/users';; 
const app = express(); 
app 
.get(/hello/, r => r.res.end('Hello, world!')) 
.get('/login', r => r.res.render('login'))
.get(/users/, async r => r.res.end('list', { title: 'Список логинов', items})) 
.use(r => r.res.status(404).end('Still not here, sorry!')) 
.use((e, r, res, n) => res.status(500).end('Error: ${e}')) 
.set('view engine', 'pug') 
.listen(process.env.PORT || PORT, async () => { 
console.log(`Старт процесса: ${process.pid}`); 
//items = (await get(URL)).data.users; 
({ data: {users: items } } = await get(URL)); 
});
```
### 2. Для установки компонента Express, отвечающего за обработку данных, посланных методом post, выполняем `yarn add body-parser`
### 3. Добавляем в код соответствующую зависимость `const bodyParser = require('body-parser')` и ее использование в приложении `.use(bodyParser.json())` , `.use(bodyParser.urlencoded({ extended: true }))`
![](https://github.com/arinasaf11/express-session170418/blob/master/Screenshot_5.png)
### 4. Выполняем `yarn add express-session` для установки компонента Express, отвечающего за обработку сессий
### 5. Рефакторинг кода  
Добавляем в код еще одну зависимость `const session = require('express-session')` и ее использование `.use(session({ secret:	'mysecret',	resave:	true,	saveUninitialized:	true }))`. Затем добавляем запись данных сессии в ветку проверки наличия пользователя,
соответствующую успешной проверке совпадения найденного пользователя и его пароля: `req.session.auth = 'ok';`  
Добавляем функцию checkAuth промежуточного программного обеспечения (конвейера, middleware) для обработки защищённых маршрутов и для защиты маршрута /users.  
Добавляем маршрут /logout для прекращения сессии.
```javascript
const express = require('express');
const { get } = require('axios');
const bodyParser = require('body-parser');
const session = require('express-session');

let items;
const PORT = 4321;
const URL = 'https://kodaktor.ru/j/users';
const app = express();
const checkAuth = (r, res, next) => { //функция checkAuth
	if (r.session.auth === 'ok') {
		next();
	} else {
		res.redirect('/login');
	}
};
app
  .use(bodyParser.json())
  .use(bodyParser.urlencoded({ extended: true }))
  .use(session({	secret:	'mysecret',	resave:	true,	saveUninitialized:	true }))

  .get(/hello/, r => r.res.end('Hello world!'))
  .get('/login', r => r.res.render('login'))
  .post('/login/check', r => {
  	const { body: { login: l } } = r;
  	const user = items.find(({ login }) => login === l);
  	if (user){
  		if (user.password == r.body.pass){
  			r.session.auth = 'ok';
  			//r.res.send('Fine!');
  			r.res.redirect('/users');
  		} else {
  			r.res.send('Wrong pass!');
  		}
  	} else {
  		r.res.send('No such user!');
  	}
  })
  .get('/logout', r => {
  	r.session.auth === null;
  	r.res.redirect('/login');
  })
  .get(/users/, checkAuth, async r => r.res.render('list', { title: 'Список логинов', items }))
  .get('/author', r => 
  	r.res.send('<h4 id="author" title="GossJS">Sophia Kramar</h4>')
  )
  .use(r => r.res.status(404).end('Still not here, sorry!'))
  .use((e, r, res, n) => res.status(500).end(`Error: ${e}`))
  .set('view engine', 'pug')
  .listen(process.env.PORT || PORT, async () => {
  	console.log(process.pid);
  ({ data: {users: items} } = await get(URL));
});
```
### 6. Самостоятельно добавляем в папку views файл login.pug, содержащий шаблон страницы для ввода логина и пароля (если команда `curl -kSL 'https://kodaktor.ru/g/session_pug' -o './views/login.pug'` вызывала ошибку)
### 7. Проверяем, чтобы попытка посетить маршрут /users без предварительного залогинивания приводила к перемещению к маршруту /login, после успешного заполнения формы по которому верными логином и паролем повторное посещение маршрута /users будет уже успешным, и так до посещения маршрута /logout или до окончания срока сессии.
![](https://github.com/arinasaf11/express-session170418/blob/master/Screenshot_login.png?raw=true)
![](https://github.com/arinasaf11/express-session170418/blob/master/Screenshot_users.png?raw=true)
