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
<br/>

![https://github.com/endansdto/my-ctf-challenges/blob/main/KAPO%202024/JSFull/20240923_030206.png](https://github.com/endansdto/my-ctf-challenges/blob/main/KAPO%202024/JSFull/20240923_030206.png)
* Chrome의 모든 Connection Pool(256개)을 Block합니다.
* `/secret/0`을 open하고 `//a.com`을 fetch합니다.
<br/>

![https://github.com/endansdto/my-ctf-challenges/blob/main/KAPO%202024/JSFull/20240923_030235.png](https://github.com/endansdto/my-ctf-challenges/blob/main/KAPO%202024/JSFull/20240923_030235.png)
* 4번째로 1개의 소켓을 열면 `/secret/0`와 `//a.com` 요청이 순서대로 1개의 소켓을 이용하여 이루어집니다.
* 이때 `//a.com` 서버에서는 Keep-Alive를 활성화해야 합니다.
* 활성화되었을 경우, `//a.com` 요청 이후의 소켓 1개는 `//a.com`과 TCP 연결이 끊기지 않고 연결됩니다.
<br/>

![https://github.com/endansdto/my-ctf-challenges/blob/main/KAPO%202024/JSFull/20240923_030246.png](https://github.com/endansdto/my-ctf-challenges/blob/main/KAPO%202024/JSFull/20240923_030246.png)
* `/secret/0`의 HTML을 분석한 Chrome은 img가 존재할 경우 새로운 요청을 생성해야 합니다.
* 이때 255개의 소켓은 모두 Block이기 때문에 Keep-Alive를 통해 TCP 연결이 되어 있는 1개의 소켓이 강제로 끊어집니다.
* 소켓이 끊어졌다는 사실은 `a.com` 서버에서 확인할 수 있으므로 img가 존재하는지 존재하지 않는지 판별할 수 있습니다.
* secret을 유출한 다음 Flag를 획득합니다.
