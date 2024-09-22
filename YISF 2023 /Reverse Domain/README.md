# Reverse Domain

* Category: Web
* Difficulty: Medium

## Description

rDNS를 구현하기 위해 Ivy만의 테스트 서버를 만들었어요.

## Backup

[https://dreamhack.io/wargame/challenges/908](https://dreamhack.io/wargame/challenges/908)

## Simple Solution

문제의 접근 방식은 대략 아래와 같습니다.
1. `isUpperCase(req.method)`가 `false`인 HTTP Method를 찾습니다.
2. 주어진 임의의 아이피 100개에 맞는 도메인 100개를 3초 이내로 찾습니다.

### Note

```javascript
function isUpperCase(str) {
  return /^[A-Z]+$/.test(str);
}

app.use(async (req, res, next) => {
	if (!isUpperCase(req.method)) {
    ...
```
* Express의 `req.method`는 HTTP Method를 대문자로 반환합니다. GET 요청의 경우 `GET`을 반환합니다.
<br/>
  
![https://github.com/endansdto/my-ctf-challenges/blob/main/YISF%202023%20/Reverse%20Domain/20240922_180856.png](https://github.com/endansdto/my-ctf-challenges/blob/main/YISF%202023%20/Reverse%20Domain/20240922_180856.png)
[https://expressjs.com/en/5x/api.html#routing-methods](https://expressjs.com/en/5x/api.html#routing-methods)
* Express 문서를 확인하여 `m-search`를 찾고 해당 HTTP Method로 요청합니다.
<br/>

```javascript
const domains = req.query?.domains;
if (!(Array.isArray(domains) && domains?.length === 100 && domains.every(d => typeof d === 'string')))
  return res.send("check ur domains");

let real_ips;
try {
  real_ips = await Promise.all(domains.map(dns.resolve4));
} catch (e) {
  return res.send("cheer up !");
}

if (real_ips.every((real_ip, i) => real_ip[0] === req.session.ips[i]))
  return res.send(FLAG);
```
* [https://nip.io/](https://nip.io/)와 같은 서비스를 이용하면 `123.123.123.123.nip.io` -> `123.123.123.123` 처럼 아이피에 대한 Domain을 쉽게 찾을 수 있습니다.
<br/>

## Unintended Solution
* 직접 DNS Server를 구축하여 아이피에 대한 Domain을 생성하여 풀이한 참가자가 존재했습니다.
  
