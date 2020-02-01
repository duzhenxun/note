### go代码
```golang
package main

import (
	"crypto/hmac"
	"crypto/sha256"
	"encoding/base64"
	"encoding/json"
	"fmt"
	"go-demo/01/jwt"
	"strings"
)

func main() {
	//jwt.io 密钥 123456 注意payload的字符串要与官网一样
	payload:=map[string]interface{}{"key":"ado"}
	secret:="123456"
	p, _ := json.Marshal(payload)
	token:=jwtEncode(string(p),secret)
	fmt.Println(token)

	t, _ := jwt.Encode(payload, []byte(secret), "HS256")
	fmt.Println(string(t))
}

func jwtEncode(payload string, secret string) string {
	header:=`{"alg":"HS256","typ":"JWT"}`
	segments := [3]string{}
	segments[0] = base64url_encode(string(header))
	segments[1] = base64url_encode(payload)

	sha := hmac.New(sha256.New, []byte(secret))
	s:=strings.Join(segments[:2], ".")

	sha.Write([]byte(s))
	segments[2] = base64url_encode(string(sha.Sum(nil)))

	return strings.Join(segments[:], ".")

}

func base64url_encode(b string) string {
	encoded := base64.URLEncoding.EncodeToString([]byte(b))
	var equalIndex = strings.Index(encoded, "=")
	if equalIndex > -1 {
		encoded = encoded[:equalIndex]
	}
	return encoded
}
```
### PHP
```PHP
<?php

/**
 * PHP jwt的实现
 * @author Zhenxun<5552123@qq.com>
 */
class Jwt
{

    //头部
    private static $header = array(
        'alg' => 'HS256', //生成signature的算法
        'typ' => 'JWT'  //类型
    );

    //使用HMAC生成信息摘要时所使用的密钥
    private static $key = '123456';

    public static function getToken(array $payload)
    {
        $base64header = self::base64UrlEncode(json_encode(self::$header, JSON_UNESCAPED_UNICODE));
        $base64payload = self::base64UrlEncode(json_encode($payload, JSON_UNESCAPED_UNICODE));
        $token = $base64header . '.' . $base64payload . '.' . self::signature($base64header . '.' . $base64payload, self::$key, self::$header['alg']);
        return $token;

    }

    private static function base64UrlEncode(string $input)
    {
        return str_replace('=', '', strtr(base64_encode($input), '+/', '-_'));
    }

    private static function signature(string $input, string $key, string $alg = 'HS256')
    {
        $alg_config = array(
            'HS256' => 'sha256'
        );
        return self::base64UrlEncode(hash_hmac($alg_config[$alg], $input, $key, true));
    }
}

//eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJhZ2UiOjIwLCJuYW1lIjoiZHV6aGVueHVuIn0.Ddh8knTN5SgLFuk4_04sijv8i906xtzGpjkhJEaaBcA%
$payload = array('age' => 20, 'name' => 'duzhenxun');
$jwt = new Jwt;
$token = $jwt->getToken($payload);
echo $token;

```
