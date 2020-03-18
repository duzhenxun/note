laravel的validator用的是 \Illuminate\Support\Facades\Validator 这个类
#### 第一种方式
```php
 $params = request(['name', 'password','xxx']);
        $roules = [
            [
            'name' => 'required|max:25',
            'password'=>'required',
             //自定义规则
            'xxx'=>function($attr,$value,$fail){
                if($value!=1){
                    $fail(sprintf("%s 必须是1",$attr));
                }
            }],
        ];
        $validatedData = \Illuminate\Support\Facades\Validator::make($params,$roules,['name.required'=>'xxxxx'],['name'=>'用户名','password'=>'密码']);
        $validatedData->validated();
        $validatedData->failed();
        $validatedData->getMessageBag()->first();

```

### 第二种方式
```php
 $params = request(['name', 'password','xxx']);
        $roules = [
            [
            'name' => 'required|max:25',
            'password'=>'required',
             //自定义规则
            'xxx'=>function($attr,$value,$fail){
                if($value!=1){
                    $fail(sprintf("%s 必须是1",$attr));
                }
            }],
	
		['name.required'=>'xxxxx'],
		['name'=>'用户名','password'=>'密码']
        ];
        $validatedData = \Illuminate\Support\Facades\Validator::make($params,...$roules);
        $validatedData->validated();
        $validatedData->failed();
        $validatedData->getMessageBag()->first();
```

