# MBTIvy Service !

* Category: Web
* Difficulty: Hard

## Description

해커들의 성격을 분석하기 위해 MBTIvy 검사 사이트를 만들었어요.

## Backup

[https://dreamhack.io/wargame/challenges/912](https://dreamhack.io/wargame/challenges/912)

## Simple Solution

* MBTI 검사와 비슷하게 질문에 대한 답을 하면 답에 따른 유형을 볼 수 있습니다.
* XS-Leak으로 Bot이 선택한 답을 구할 수 있습니다.
* 앞쪽 답과 뒤쪽 답에 대한 Bot의 차이가 존재하는데 서로 다른 XS-Leak을 의도했었습니다.
* 목표는 Bot이 랜덤하게 선택한 답들을 찾고 댓글에 존재하는 Flag를 획득하는 것입니다.

### Note

```python
@app.after_request
def add_header(res):   
    if 'nonce' in g:
        res.headers['Content-Security-Policy'] = f"default-src 'none'; script-src 'nonce-{g.nonce}' 'unsafe-inline'; connect-src 'self'; style-src 'self';"
    else:
        res.headers['Content-Security-Policy'] = f"default-src 'none'; connect-src 'self'; style-src 'self';"
    res.headers['Cross-Origin-Embedder-Policy'] = 'require-corp'
    res.headers['Cross-Origin-Resource-Policy'] = 'same-origin'
    res.headers['X-Content-Type-Options'] = 'nosniff'
    res.headers['X-Frame-Options'] = 'deny'
    res.headers['Document-Policy'] = 'force-load-at-top'
    res.headers['Referrer-Policy'] = 'no-referrer'
    res.headers['Cache-Control'] = 'max-age=60'
    return res
```
* `Cache-Control: max-age=60`에서 Cache를 사용하는 것을 확인합니다.
<br/>

```python
def bot(url):
  URL = f"http://mbtivy.kro.kr"
  driver.get(URL)
  wait3.until(EC.presence_of_element_located((By.TAG_NAME, 'body')))
  
  for i in range(15 + 1):
      buttons = driver.find_elements(By.CLASS_NAME, 'check-button')
      secrets.choice(buttons).click()
      wait3.until(EC.presence_of_element_located((By.TAG_NAME, 'body')))
```
* Bot은 `http://mbtivy.kro.kr`로 이동하여 15개의 질문에 대한 랜덤한 답을 합니다.
* URL에 대한 요청 시간을 측정한다면 Caching 여부를 확인할 수 있습니다.
* Caching이 되었다면 봇이 선택했던 답임을 확신할 수 있습니다.
<br/>

![https://github.com/endansdto/my-ctf-challenges/blob/main/YISF%202023/MBTIvy%20Service%20!/20240922_191018.png](https://github.com/endansdto/my-ctf-challenges/blob/main/YISF%202023/MBTIvy%20Service%20!/20240922_191018.png)
[https://developer.chrome.com/blog/http-cache-partitioning?hl=ko](https://developer.chrome.com/blog/http-cache-partitioning?hl=ko)
* 다만 Cache Partitioning 동작을 염두해야 합니다.
* 링크를 요약하면 Chrome은 `요청하는 프레임 + 요청하는 프레임의 최상위 프레임`을 기준으로 캐싱합니다.
* Bot이 답들을 선택할 때의 Caching을 생각하면 최상위 프레임이 `http://mbtivy.kro.kr`이며 요청하는 프레임도 이와 동일할 것입니다.
* Cross-Site에서의 Caching을 생각하면 요청하는 프레임은 동일하지만 최상위 프레임이 `http://mbtivy.kro.kr`이 아니기 때문에 Caching으로 XS-Leak을 수행하기 어렵습니다.
* 자세하게 Caching 기준을 확인하면 프레임의 `scheme://eTLD+1`이 기준입니다.
* `mbtivy.kro.kr`에서 `eTLD`를 확인하면 `.kr`이 `eTLD`이며 `.kro.kr`는 `eTLD+1`입니다.
* 따라서 `asdf.kro.kr`와 `mbtivy.kro.kr`는 동일한 프레임으로 인식됩니다.
* `kro.kr` 도메인을 조사하면 알 수 있듯, `내도메인.한국` 사이트에서 무료로 `*.kro.kr` 도메인을 생성할 수 있습니다.
* `*.kro.kr` 도메인 중 하나를 생성하면 Cache Probing XS-Leak으로 봇의 첫 부분 답들을 leak할 수 있습니다.
<br/>

```python
current_url = driver.current_url
parsed_url = urlparse(current_url)
driver.get(f"http://{IP}{parsed_url.path}?{parsed_url.query}")
wait3.until(EC.presence_of_element_located((By.TAG_NAME, 'body')))

for i in range(15):
    buttons = driver.find_elements(By.CLASS_NAME, 'check-button')
    secrets.choice(buttons).click()
    wait3.until(EC.presence_of_element_located((By.TAG_NAME, 'body')))
```
* 이전과는 다르게 Domain을 이용하지 않고 IP를 이용하여 접속합니다. 따라서 Cache Probing을 사용하기는 어렵습니다.
* Terjanq의 [XS-Leak Wiki](https://xsleaks.dev/)에서 [적당한 기법](https://xsleaks.dev/docs/attacks/css-tricks/)을 찾을 수 있습니다.
    * `<a>` 태그에서 `href`를 변경하면 `:visited` 여부에 따라 파란색/보라색으로 색상이 변경됩니다.
    * `<a>` 태그의 innerText를 매우 크게(최소 몇 만 이상) 잡으면 색상이 변경될 때 어느 정도의 딜레이가 생깁니다.
    * JS로 프레임이 넘어갈 때의 시간 차이를 측정하면 색상이 변경되었는지 판단할 수 있고 봇이 방문한 URL인지 확인할 수 있습니다.
* Visited Link를 이용한 방법은 status code가 404일 경우 동작하지 않으므로 봇의 앞쪽 답을 leak하긴 어렵습니다.
<br/>

```javascript
function clickHandler() {
    fetch('/add', { 
        headers: {
            'Content-Type': 'application/x-www-form-urlencoded'
        },
        method: "POST",
        body: `review-input=${document.getElementById('review-input').value}&select={{ select }}`
    })
    .then(res => res.text())
    .then(data => {
        if (data === 'success')
            window.location.reload()
    });
}

function parse(data) {
    const output = {};

    Object.keys(data).forEach(key => {
        const value = data[key];
        const keyParts = key.split('.');
        let currentObj = output;

        for (let i = 0; i < keyParts.length; i++) {
            const part = keyParts[i];
            if (part.indexOf('o') > 2)
                continue;
            else if (i === keyParts.length - 1) {
                currentObj[part] = () => value;
            } else {
                currentObj[part] = currentObj[part] || {};
                currentObj = currentObj[part];
            }
        }
    });

    return output;
}

function getCookie(name) {
    const value = `; ${document.cookie}`;
    const parts = value.split(`; ${name}=`);
    if (parts.length === 2) return parts.pop().split(';').shift();
}

function messageHandler(e) {
    function getHost(url) {
        try {
            return (new URL(url)).host;
        } catch (err) {}
    }

    if (getHost(e.origin) !== CHECK.host || e.source != CHECK.source || getHost(CHECK.is_validate) === 'true')
        return;
        
    const data = parse(e.data);
    if (getCookie('develop_hash') !== '45a29cef0f0a8a0c06f6244edc474be1ecd757a070d1ea9a982410a630b34908')
        return;	
        
    const develop_url = data.url();
    if (/^[\x00-\x7F]*$/.test(develop_url) && develop_url[0] !== '/')
        window.location = '/' + develop_url + encodeURIComponent(btoa(review.join()));
    
    
}

review = {{ review|tojson }};
review_div = document.getElementsByClassName('review')[0];
window.review.map(r => {
    const div = document.createElement("div");
    div.classList.add("message");
    if (window.Sanitizer)
        div.setHTML(r);
    else
        div.innerHTML = DOMPurify.sanitize(r)
    review_div.appendChild(div);
});

window.onmessage = (e) => setTimeout(() => messageHandler(e), 150);
document.getElementById("review-button").addEventListener("click", clickHandler);

CHECK = {host: "localhost", source: window, is_validate: 'http://true'};
```
* 댓글을 JSON으로 받아와 `Sanitizer API` 또는 `DOMPurify`로 살균한 뒤 추가합니다. (Chrome Bot의 버전이 127이기 때문에 Sanitizer API를 사용할 수 있습니다)
* `onmessage`를 통해 특정 메시지를 전송할 수 있습니다.
* `Sanitizer API`는 Dom Clobbering을 방어하지 않기 때문에 `getElementById`를 덮어쓴다면 오류가 발생하여 `CHECK` 변수가 초기화되지 않습니다.
* `e.origin`은 요청 페이지가 sandbox iframe일 경우 `null`이기 때문에 우회할 수 있습니다.
* `e.source`는 요청 페이지가 바로 닫힐 경우 원본 객체를 잃어버리기 때문에 우회할 수 있습니다.
* `CHECK.is_validate`는 Dom Clobbering으로 조건에 맞는 값을 채울 수 있습니다.
* `develop_hash`를 설정하기 위해서는 일반적으로 cookie의 접근이 필요해보이나 우회가 가능합니다.
    * Dom Clobbering으로 `document.cookie`를 덮으면 HTML 객체입니다.
    * parse 함수에서 `constructor`와 `prototype`으로 `prototype pollution`이 발생합니다.
    * `Object.toString`을 `() => 'a=hello'`로 덮게 되면 `getCookie('a')`는 `hello`를 반환합니다.
* `develop_url`를 적당하게 세팅하면 임의의 URL로 Flag가 담긴 댓글이 전송됩니다. 

## Unintended Solution 

* 매우 큰 크기의 페이지를 생성하거나 http 요청을 멈추는 등의 방식을 통해 Bot의 Timeout을 늘릴 수 있습니다.
    * DNS Rebinding을 사용하여 XS-Leak을 수행한 이후의 과정을 우회할 수 있습니다
    * DNS Rebinding은 워게임에서 막아두었으나 Bot Timeout은 막지 않았습니다.
* Cache probing + Visited Link을 이용하지 않고 Connection Pool + Error를 이용하여 봇의 답들을 leak할 수 있습니다.
    * Chrome의 Connection Pool을 모두 막고 `w=open(...)`으로 leak하고 싶은 URL을 엽니다.
    * `w.origin`이 오류를 일으킨다면 Cache가 동작한 것이므로 봇이 방문했던 URL이며, 오류를 일으키지 않는다면 Caching되지 않은 것이므로 봇이 방문하지 않은 URL입니다.
    * 이때 Cache Partitioning은 최상위 프레임과 요청 프레임 모두 `http://mbtivy.kro.kr`이기 때문에 정상적으로 동작합니다.
    * 인텐 풀이와 차이는 존재하지만 창의적이고 충분히 난이도 있다 판단하여 워게임에서 막아두지 않았습니다.
  
