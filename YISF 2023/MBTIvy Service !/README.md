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
* Bot은 `http://mbtivy.kro.kr`로 이동하여 16개의 질문에 대한 랜덤한 답을 합니다.
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



## Unintended Solution 
  
