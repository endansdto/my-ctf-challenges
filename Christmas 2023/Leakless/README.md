# Leakless

* Category: Web
* Difficulty: Hard

## Description

CSP도 존재하지 않고 alert(1)도 가능하네요.
그럼 플래그만 얻으면 되는 거죠...?

## Backup

[https://dreamhack.io/wargame/challenges/1056](https://dreamhack.io/wargame/challenges/1056)

## Simple Solution

* 회원가입, 로그인, 글쓰기 기능이 존재합니다.
* 글은 작성자의 checked를 true로 변경 후 볼 수 있습니다.
* 봇이 작성한 글에 존재하는 FLAG를 읽어야 합니다.

### Note

```javascript
app.post('/register', (req, res) => {
    const username = req.body.username + '';
    const password = req.body.password + '';
    const country = req.body.country + ''
	
	if (username.length > 0 && password.length > 0 && country.length > 0 && username != ADMIN_ID) {
		const user = {password: password, country: country, content: 'hello', viewtime: dayjs(getRandomDateTime()), checked: false};
		accounts[username] = user;
		res.send(`<script>location.href='/login';</script>`);
	} else {
		res.send(`<script>alert('fail');history.back();</script>`);
	}
});
```
* 회원가입 시 checked: false입니다.
* viewtime은 랜덤으로 초기화됩니다.
<br/>

```javascript
app.get('/check', (req, res) => {
	if (req.session.username && accounts[req.session.username].viewtime.format('YYYY-MM-DD HH:mm:ss') === req.query?.day + '') {
		accounts[req.session.username].checked = true;
		res.send(`<script>history.back();</script>`);
	} else {		
		res.send(`<script>alert('fail');history.back();</script>`);
	}
});
```
* viewtime을 알아내면 checked를 true로 설정할 수 있습니다.
<br/>

```javascript
const isAdmin = (req, res, next) => {
	if (req.session.username !== ADMIN_ID)
         res.send(`<script>alert('now allowed');history.back();</script>`);
	next();
};

app.get('/clear', isAdmin, (req, res) => {
	if (req.session?.username === 'clear_account') {
		accounts = {};
		ADMIN_PS = crypto.randomBytes(30).toString('hex');
		accounts[ADMIN_ID] = {'password': ADMIN_PS, country: 'ko', content: 'hello', viewtime: dayjs(), checked: true};
		res.send(`<script>history.back();</script>`);
	} else {		
		res.send(`<script>alert('fail');history.back();</script>`);
	}
});

app.post('/write', (req, res) => {
	if (req.session && req.session.username) {
		const username = req.session.username;
		const content = req.body.content + '';
		const minutes = parseInt(req.body.minutes);
		const seconds = parseInt(req.body.seconds);
		
		if (0 <= minutes && minutes < 60 && 0 <= seconds && seconds < 60 ) {
			accounts[username].content = content;
			if (accounts[username].checked) {
				accounts[username].viewtime = dayjs().add(seconds, 'second').add(minutes, 'minute');
			}
			
			res.send(`<script>location.href='/view?username='+'${username}';</script>`);
		} else {
			res.send(`<script>alert('fail');history.back();</script>`);
		}
	} else {
		res.send(`<script>alert('fail');history.back();</script>`);
	}
});
```
* isAdmin에는 어드민인지 검증하는 코드가 존재하나 적절치 못한 코드로 인해 이후의 과정이 계속 실행됩니다.
* `clear_account` 계정을 통해 `accounts`를 초기화할 수 있습니다.
* 이때 username=`__proto__`이면 `/write`의 `accounts[username].content = content;`에서 `content` 속성을 Prototype Pollution으로 오염시킬 수 있습니다.
<br/>

```javascript
const TIME_FORMAT = {ko: 'hh:mm:ss', ch: 'hh:mm:s', jp: 'hh:m:s'};
app.get('/view', (req, res) => {
	const username = req.query.username + '';
	
	if (username in accounts && (username != ADMIN_ID || (req.session?.username === ADMIN_ID))) {
		const user = accounts[username];
		if (user.checked && dayjs().isAfter(user.viewtime)) {
			//it looks free xss. right?
			res.send(`<button id="logoutButton" onClick="location.href='/logout'">로그아웃</button>${user.content}`);	
		} else {
			const until = user.viewtime.format(TIME_FORMAT[user.country]);
			res.send(`<script>alert('wait until ${until}');</script>`);	
		}
	} else {
		res.send(`<script>alert('fail');history.back();</script>`);
	}
});
```
* `user.country`는 임의로 설정할 수 있습니다.
* `user.country = 'content'`이면 `TIME_FORMAT.content`를 참조하게 되어 PP로 오염된 값을 가져옵니다.
* 따라서 `user.viewtime.format`에 대한 인자를 임의로 줄 수 있고 viewtime을 알아낼 수 있습니다.
<br/>

```javascript
app.get('/check', (req, res) => {
	if (req.session.username && accounts[req.session.username].viewtime.format('YYYY-MM-DD HH:mm:ss') === req.query?.day + '') {
		accounts[req.session.username].checked = true;
		res.send(`<script>history.back();</script>`);
	} else {		
		res.send(`<script>alert('fail');history.back();</script>`);
	}
});
```
* `/clear`에서 알아낸 viewtime을 통해 checked를 true로 설정할 수 있습니다.
<br/>

```javascript
browser = await puppeteer.launch({
    headless: 'new',
    executablePath: '/usr/bin/google-chrome',
    args: ['--no-sandbox', '--disable-dev-shm-usage', '--disable-gpu', '--incognito', '--js-flags=--noexpose_wasm,--jitless',]
});

let page = await browser.newPage();

await page.goto('http://localhost/login', { timeout: 1000, waitUntil: 'domcontentloaded' });
await page.evaluate((ADMIN_ID, ADMIN_PS) => {
    document.querySelector('#username').value = ADMIN_ID;
    document.querySelector('#password').value = ADMIN_PS;
    document.querySelector("#submit").click();
}, ADMIN_ID, ADMIN_PS);
await page.waitForTimeout(1000);

await page.evaluate((FLAG) => {
    document.querySelector('#content').value = FLAG;
    document.querySelector("#submit").click();
}, FLAG);
await page.waitForTimeout(1000);

await page.evaluate(() => {
    document.querySelector("#logoutButton").click();
});
await page.waitForTimeout(1000);

let page2 = await browser.newPage();
await page.close();

await page2.goto(url, { timeout: 5000, waitUntil: 'domcontentloaded' })
await page2.waitForTimeout(5000);

await page2.close();
await browser.close();
```
* Bot을 관찰하면, `로그인 -> 게시글 작성 -> 로그아웃 -> URL 이동` 순서로 동작합니다.
* 로그아웃을 할 경우 로그인 정보가 남아있지 않기 때문에 XSS를 발생시켜도 무언가 기대하기는 어렵습니다.
* 이 부분이 메인 아이디어이고 사실 이 아이디어를 위해 만든 문제가 `Leakless`입니다.
<br/>

* Bot의 코드를 확인하면 로그아웃 버튼을 클릭한 뒤 단순하게 1초를 기다리고 다음 페이지를 엽니다.
* 로컬 서버이긴 해도, 만약 서버를 잠시 동안 멈출 수 있다면 로그아웃을 막을 수 있지 않을까요?
* 서버는 Node.js로 동작하며 이는 잘 알려졌다시피 단일 스레드를 기반으로 하고 있습니다. 무언가 기대해볼 수 있지 않을까요?
* 위의 과정 중에서 dayjs의 `format` 함수의 인자로 임의의 값을 넘겨주었던 것을 생각할 수 있습니다.
<br/>

```javascript
  format(formatStr) {
    const locale = this.$locale()
    if (!this.isValid()) return locale.invalidDate || C.INVALID_DATE_STRING
    const str = formatStr || C.FORMAT_DEFAULT

    ...

    return str.replace(C.REGEX_FORMAT, (match, $1) => $1 || matches(match) || zoneStr.replace(':', '')) // 'ZZ'
  }
```
```javascript
export const REGEX_FORMAT = /\[([^\]]+)]|Y{1,4}|M{1,4}|D{1,2}|d{1,4}|H{1,2}|h{1,2}|a|A|m{1,2}|s{1,2}|Z{1,2}|SSS/g
```
* dayjs의 format 함수를 살펴보면 마지막 리턴 부분에서 `c.REGEX_FORMAT`를 사용합니다.
* 사용된 정규식을 살펴보면 `\[([^\]]+)]`라는 정규식을 부분적으로 사용하는데 이는 특정 입력에 대해 O($$n^2$$)의 시간복잡도를 갖습니다.
<br/>

```javascript
app.use(express.urlencoded({
	limit: '20kb',
	extended: false
}));
```
* 문제에서 설정된 HTTP Limit를 살펴보면 20kb 정도임을 확인할 수 있는데, 20kb의 입력으로 O($$n^2$$)의 시간복잡도를 적용하면 약 1~2초 정도 지연이 발생합니다.
* 따라서 ReDoS를 일으켜 Bot의 로그아웃을 방어할 수 있고 간단한 XSS를 통해 Flag를 유출할 수 있습니다.
  

## Unintended Solution
* 대회 당시에는 `back/forward cache`를 생각지 못했습니다. `Cache-Control: no-store`이어도 `bfcache`는 동작하는 듯합니다. 워게임에서는 수정되었습니다.
* [검증되지 않음] 최근에 발견된 Node.js DoS CVE로 시도해도 Bot의 로그아웃을 방어할 수 있을 것 같습니다.
