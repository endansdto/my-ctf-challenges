# JSLess

* Category: Web
* Difficulty: Hard

## Description

풀이에 성공한 이들에게는 수많은 풀이꾼들이 그토록 바랐던 GOSU의 FLAG가 증표처럼 남겨진다.

## Backup

Dreamhack.io 포팅 예정

## Simple Solution

* \<img\> 또는 text가 존재하는 페이지를 구분하는 XS-Leak 문제입니다.


### Note
```javascript
app.use((req, res, next) => {
    res.setHeader("Content-Security-Policy", "default-src 'self'; frame-ancestors 'none';");
	res.setHeader("Cross-Origin-Embedder-Policy", "require-corp");
    res.setHeader("Cross-Origin-Opener-Policy", "same-origin");
    res.setHeader("Cross-Origin-Resource-Policy", "same-origin");
    res.setHeader("X-Frame-Options", "deny");
    res.setHeader("X-Content-Type-Options", "nosniff");
    res.setHeader("Cache-Control", "no-store");
    res.setHeader("Document-Policy", "force-load-at-top");
    res.setHeader("Referrer-Policy", "no-referrer");
	
	next();
});
```
* CSP가 self로 잡혀있고 Cache는 no-store입니다.
<br/>

```javascript
let secret = Array.from(crypto.randomBytes(40), byte => byte % 2);

app.get('/secret/:index', (req, res) => {
	if (!(/^[0123456789]+$/.test(req.params.index ?? '')))
		return res.send('Invalid index');
	const index = parseInt(req.params.index ?? '');
	if (index < 0 || index >= secret.length)
		return res.send('Invalid index');
	
	if (req.cookies?.bot_token !== bot_token)
		return res.send(index % 2 ? `<img src='/gosu.webp'>` : 'i wanna gosu');
	
	secret[index] = crypto.randomInt(2);
	res.send(secret[index] ? `<img src='/gosu.webp'>` : 'i wanna gosu');
});
```
* secret은 [1, 1, 0, 1, 0, 0, 1, ...]과 같이 랜덤한 이진 배열입니다.
* `/secret/0/`으로 접속하면 `secret[0]`의 값에 따라 하나의 `<img src=gosu.webp>`가 보이거나 `i wanna gosu`가 보입니다.
* 확인하기 직전의 인덱스는 항상 초기화됩니다.
* 특정 페이지에 img가 존재하는지 존재하지 않는지만 판별하면 됩니다.



