### golang
```go
func hmacSha256(src string, secret string) string {
	h := hmac.New(sha256.New, []byte(secret))
	h.Write([]byte(src))
	shaStr:= fmt.Sprintf("%x",h.Sum(nil))
	//shaStr:=hex.EncodeToString(h.Sum(nil))
	return base64.StdEncoding.EncodeToString([]byte(shaStr))
}
fmt.Println(hmacSha256("hello", "duzhenxun"))

```
### PHP
```PHP
echo base64_encode(hash_hmac('sha256','hello','duzhenxun'));

```

