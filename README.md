# Easily-master-GO-reflect

轻松掌握GO反射 Easily master GO reflect

## 开篇

本次我们将以 两个练习 带领 让大家来掌握golang的反射


### 练习1 实现一个简易的 validator

```go
type LoginRequest struct {
	Email     string `json:"email" validator:"required"`
	Password string `json:"password" validator:"required"`
}

练习1 我们需要 验证 LoginRequest  email 和 password 字段是否为空

func Validate(input interface{}) bool {
	refType := reflect.TypeOf(input)   // 得到 入参 类型相关信息
	refValue := reflect.ValueOf(input) // 得到 入参 值相关信息

	fmt.Println(refType.Kind()) // 打印 input 类型
	if refType.Kind() == reflect.Ptr {
		// 如果当前输入是一个指针
		// 我们 就取 里面的具体类型
		refType = refType.Elem()
		refValue = refValue.Elem()
	}
	fmt.Println(refType.Kind()) // 打印 input 具体 类型

	numField := refType.NumField()  // 获取里面字段具体个数
	for i := 0; i < numField; i++ { // 遍历这个字段
		field := refType.Field(i) // 按照 下标 获取字段
		tag := field.Tag.Get("validator")

		switch tag {
		case "required": // 处理具体事件
			value := refValue.Field(i) // 更具下标 获取具体值
			switch field.Type.Kind() { // 更具字段不同的类型 编写不同的 required 处理逻辑
			case reflect.String:
				if len(strings.TrimSpace(value.String())) == 0 {
					return false
				}
			case reflect.Int16 | reflect.Int32:
				// TODO ... 更具字段不同的类型 编写不同的 required 处理逻辑
			}
		}
	}

	return true
}
```


### 本节所学: 

``` 
	refType := reflect.TypeOf(input)   // 得到 入参 类型相关信息
	refValue := reflect.ValueOf(input) // 得到 入参 值相关信息
	refType.Kind()  // 获取当前 refType 的具体类型
	refType.Elem()  // refType 取 地址, 注意如果 当前 refType 不是 指针 会 painc
	numField := refType.NumField()  // 获取 struct 中字段个数
	
	field := refType.Field(i) // 按照 下标 获取字段
	tag := field.Tag.Get("validator")  // 获取字段的tag
	
	value := refValue.Field(i) // 更具下标 获取refValue具体值
```


### 练习2 `map[string]interface` 转 `struct`

```go
type Request2 struct {
	Email    string  `json:"email" validator:"required"`
	Password string  `json:"password" validator:"required"`
	Name     *string `json:"name"`
	Age      int64   `json:"age"`
}

func TestReflect2(t *testing.T) {
	var output Request2

	input := map[string]interface{}{
		"email":    "dollarkiller@dollarkiller.com",
		"password": "this is password",
		"name":     "dollarkiller",
		"age":      18,
	}

	Convert(input, &output)
}

func Convert(input map[string]interface{}, output interface{}) {}
```


演示代码 写的不太优雅 请君见谅
```go
func Convert(input map[string]interface{}, output interface{}) error {
	// 第一步 我们 先获取基础信息
	refType := reflect.TypeOf(output)
	refValue := reflect.ValueOf(output)

	// 我们在做参数绑定 必然 output 需要是一个指针类型
	if refType.Kind() != reflect.Ptr {
		return errors.New("output is not ptr")
	}
	refType = refType.Elem() // 我们取一层地址

	// 第二步 遍历 这个 结构体
	numField := refType.NumField()
	for i := 0; i < numField; i++ {
		field := refType.Field(i)            // 我们一个一个字段去操作
		valField := refValue.Elem().Field(i) // 值操作
		if field.IsExported() {              // 是否是public字段
			jsonTag := field.Tag.Get("json") // 更具 json tag 来作为 填充映射关系

			// 更具不同类型去填充
			switch field.Type.Kind() {
			case reflect.String:
				val, ok := input[jsonTag]
				if ok {
					s, ok := val.(string)
					if ok {
						valField.SetString(s)
					}
				}
			case reflect.Int64:
				val, ok := input[jsonTag]
				if ok {
					s, ok := val.(int) // interface 中存在 int， 断言会默认识别为 int
					if ok {
						valField.SetInt(int64(s))
					}
				}
			case reflect.Ptr:
				// 这里用递归写法会很舒服 ， 作为教学教程这里就按照清晰易懂得写法
				newValue := reflect.New(field.Type.Elem()) // 取地址然后新建  !!!
				tType := newValue.Type()
				switch tType.Elem().Kind() {
				case reflect.String:
					val, ok := input[jsonTag]
					if ok {
						s, ok := val.(string)
						if ok {
							newValue.Elem().SetString(s)
							valField.Set(newValue)
						}
					}
				case reflect.Int32 | reflect.Int16 | reflect.Int64:
					// TODO ...
				}

			}
		}
	}

	return nil
}
```


