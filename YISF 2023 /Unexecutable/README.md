# Unexecutable

* Category: Web
* Difficulty: Hard

## Description

실행불가한 .

## Backup

[https://dreamhack.io/wargame/challenges/916](https://dreamhack.io/wargame/challenges/916)

## Simple Solution

문제의 접근 방식은 대략 아래와 같습니다.
1. `/upload.php`로 적절한 파일명과 필터를 변수로 넘겨주어 적당한 파일을 `/var/www/html/files`에 생성합니다.
2. 1에서 생성된 URL로 Bot을 접근시켜 Flag를 획득합니다.

### Note

```php
$name = $_POST['name_name name'];
```

* PHP는 공백과 같은 몇 특수문자가 들어오면 이를 `_`로 바꿔버리는 특징이 존재합니다.
* 문제에서 사용되는 `Dockerfile`을 확인하면 `php7.4`를 사용합니다.
* 해당 버전에서는 정확한 파싱 과정을 따르지 않아 뒷 과정에서 `_`로 변환하는 과정이 생략될 수 있어 공백을 포함시킬 수 있습니다.
<br/>

```php
if (strlen($filter) > 1200 || strpos($filter, '.') !== false || strpos($filter, '/') !== false
    || strpos(strtolower($filter), '%2e') !== false || strpos(strtolower($filter), '%2f') !== false)
    die('<script>alert("invalid filter");history.back();</script>');
$img = file_get_contents("php://filter/$filter/resource=basedpng.txt");

if (strpos($img, 'resource') !== false)
    die('<script>alert("invalid filter");history.back();</script>');
$img = base64_decode($img);

if (!isset($_SESSION['path']))
    $_SESSION['path'] = bin2hex(random_bytes(20));
$path = "files/" . $_SESSION['path'];

if (!is_dir($path))
    mkdir($path);

file_put_contents("$path/$name", $img);
```

* `basedpng.txt`에서 사진 데이터를 읽은 뒤 `$filter`를 적용하여 파일로 저장합니다.
* 인코딩 필터링을 이용한 임의 데이터 생성은 크기 제한으로 쉽진 않습니다.
* 해당 PHP 버전만 적용되는지는 확인이 필요한데 놀랍게도 PHP Wrapper는 `/resource=...` 이후에 `|` 문자가 존재하면 이를 필터 구분자로 인식합니다.
* 예를 들어 `php://filter/convert.base64-encode/resource=a.png|convert.base64-encode`로 작성하면 맨 뒤의 `convert.base64-encode`는 파일명으로 인식이 되어야 할 것 같지만 실제로는 필터 동작이 수행됩니다.
* 추가적으로 PHP에서 제공하는 dechunk 필터를 이용하여 임의 데이터를 생성할 수 있습니다.
<br/>

```php
function check_valid($str) {
  $len = strlen($str);
  for ($i = 0; $i < $len; $i++) {
    if (strpos("abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789", $str[$i]) === false)
      return False;
  }

  return True;
};

if (!check_valid($name) || strlen($name) > 10)
  die('<script>alert("invalid name");history.back();</script>');
$name .= '.png';
```
* 저장되는 파일명을 확인하면 `$name + '.png'` 형태입니다.
* `.png`로 끝나게 되면 Apache에서는 이를 이미지 파일로 인식하여 Content-Type 헤더에 이미지 정보를 포함합니다.
* 웹브라우저에서는 Content-Type을 확인하고 이미지로 인식하여 XSS와 같은 공격이 어렵습니다.
<br/>

![https://github.com/endansdto/my-ctf-challenges/blob/main/YISF%202023%20/Unexecutable/20240922_181518.png](https://github.com/endansdto/my-ctf-challenges/blob/main/YISF%202023%20/Unexecutable/20240922_181518.png)
[https://x.com/YNizry/status/1582733545759330306](https://x.com/YNizry/status/1582733545759330306)

* 요약하면 Apache는 `.....png`, `.png`와 같은 파일에 대해 Content-Type 헤더를 사용하지 않습니다.
* `Content-Type`이 존재하지 않아 웹브라우저에서는 파일 타입에 대한 스니핑을 진행합니다.
<br/>

```javascript
browser = await puppeteer.launch({
  headless: 'new',
  executablePath: '/usr/bin/google-chrome',
  args: ['--no-sandbox', '--disable-dev-shm-usage', '--disable-gpu', '--js-flags=--noexpose_wasm,--jitless']
});
const page = await browser.newPage();
await page.setJavaScriptEnabled(false);
await page.goto(`http://localhost/${url}`, { timeout: 3000, waitUntil: 'domcontentloaded' });
await page.waitForTimeout(1500);

await browser.close();
```
* `setJavaScriptEnabled(false)`로 JS가 비활성화입니다. 일반적인 XSS는 어렵습니다.
* 다행히도 Chrome은 XSLT를 지원합니다. 이를 이용하여 JS 없이 Flag를 읽을 수 있습니다.
* DiceCTF에서 차용한 기법인데 2023년 정도에 Chrome에 관련한 CVE가 존재하는 것 같습니다.
<br/>

