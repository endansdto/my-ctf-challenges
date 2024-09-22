# Reverse Domain

* Category: Web
* Difficulty: Medium

## Description

올해의 X-mas를 NginX와 함께하는 분들을 위한.
NginX-mas.

## Backup

[https://dreamhack.io/wargame/challenges/1057](https://dreamhack.io/wargame/challenges/1057)

## Simple Solution

```
server {
  listen 80 default_server;
  server_name ivy.$DOMAIN;
  
  location /h {
    proxy_pass http://localhost:3333;
    proxy_set_header Host $host;
  }
}
  
server {
  listen 80;
  server_name yvi.$DOMAIN;
  
  location /f {
    proxy_pass http://localhost:3333;
    proxy_set_header Host $host;
  }
}
```
```javascript
app.get('/h', (req, res) => {
	res.json(req.headers);
});

app.get('/f', (req, res) => {
	res.send(process.env.FLAG);
});
```
* `$DOMAIN`은 알려지지 않은 값이므로 `/f`에 접근하기는 어렵습니다.
* 다행히도 `/h`는 `default_server`로 지정되어 있기 때문에 어떤 Domain을 사용하든 접근할 수 있습니다.
* Nginx는 Host Header가 전송되지 않으면 `$host`를 `server_name`의 값으로 대체합니다.
* HTTP/1.1에서는 Host Header가 필수이므로 생략 가능한 이전 버전을 사용합니다.
* Host Header를 생략하여 `/h`에 접근하면 `$DOMAIN`을 leak할 수 있습니다.
* `/f`로 접근하여 Flag를 획득합니다.

  
