---
date: "2016-07-23T14:46:01+07:00"
draft: false
title: "JSON decoding numbers"
description: "Decode arbitrary JSON data contain number"
tags: [ "golang", "json", "encode", "decode", "interface", "reflection"]
categories:
  - "Standard Library"
  - "Reflection"
---

### Preface

At previous post, we had encountered a problem when decode arbitray JSON data containing number. To recap, we decode JSON data into a `map[string]interface{}` and suffered type change, from `int` to `float64`, see more at [JSON Pitfalls]({{< relref "post/json-1.md" >}}).


This issue was mentioned at another post too, please see [50 Mistakes](http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html#json_num)

That blog author proposed these options:

1. Using the float value as-is
2. Convert the float value to the integer type you need.
3. Using a Decoder type to unmarshal JSON using `json.Number`.
4. Using a `struct` type that maps your numeric value to the numeric type you need.
5. Using a `struct` type that maps your number value to `json.RawMessage`

In my opiion, the best way to handling that should always the **option 4** which to not decode arbitrary data at all. We should know struct we are decoding to. There's other benefits of that beside handling number. We can use JSON tag to vaidate input data, so there's no more overhead to check type, check nil, empty value after that. [json.Marshal](https://golang.org/pkg/encoding/json/#Marshal)

The worst must be the **option 5** as it will break code resusability. With struct field type `json.RawMessage` our app is strongly coupled with `encoding/json`.  Using struct tag like `json`, `yaml` to declare encoding is great and we should not eliminate that.

In case we have no choice but have to decode to a `map[string]interface{}`, say it depends on other developer usage then we can consider both `option 2 & 3`. We will use decoding option `json.Number` then convert its string value to the actual number type. For generic usage, `reflection` is used to inspect all number fields.


We declare decoder like this

Our custom decoder will take input of serialized data and input interface to decode to
```
func decodeUseNumber(content []byte, in interface{}) error {
```

It has the same signature with `json.Unmarshal` function, the `Unmarshaler` interface hence we can easily switch between.

Decode using number option

```
func decodeUseNumber(content []byte, in interface{}) error {
	// decode using json number
	decoder := json.NewDecoder(bytes.NewReader(content))
	decoder.UseNumber()
	err := decoder.Decode(in)
	if err != nil {
		return err
	}
```

At this point, if no error happens, our `in` object is loaded with input serialized data. All input number will be represented by `json.Number` which is in turn a `string` named type. This `json.Number` type has useful methods to convert number string to `int` or `float`

It's ok if we stop here . But we still can do more so this function's consumer doesn't need to know about this `json.Number` and think about how to use it.

We can lookup for all arbitrary map fields contain `json.Number` type and convert them to actual number type, something like this

Pseudo Code
```
for f in struct.Fields:
    # found all arbitrary map fields
    if f.Type == map:
        for key, value in map:
            if value.Type == json.Number:
                # convert number
                map[key] = value.Integer()
```

There're two steps:

    + Find all arbitrary fields `map[string]interface`
    + Convert all `json.Number` values in map

It's better to keep theses separately developed.

We define function name `iterateMapField` which fetch all map fields and execute another function on found map

```
func iterateMapFields(v reflect.Value, fn func(map[string]interface{}) error) (err error) {
	if v.Kind() == reflect.Ptr {
		v = v.Elem() // dereference pointer
	}
	for i := 0; i < v.NumField(); i++ {
		field := v.Field(i)
		switch field.Kind() {
		case reflect.Map:
		// possibly a map[string]interface field
			m, ok := field.Interface().(map[string]interface{})
			if ok {
                        // execute function to process map here
				if err = fn(m); err != nil {
					return
				}
			}
            ...
```
More on reflection [Laws of Reflection](https://blog.golang.org/laws-of-reflection)

This method will be used like this
```
v := reflect.ValueOf(in).Elem()
iterateMapFields(v, func(mField map[string]interface{}) error {
    for k, v := range mField {
        if jsonNum, ok := v.(json.Number); ok {
        // convert value
})
```

We define an anonymous function to process map input `map[string]interface{}`.

Here is the actual conversion. As we don't have any information about the actual number type to map to, we can only guess like this. It could be improved better if we know exact type of number to be used such as `uint64`, `int8`, etc

```
// try with int first
if numInt, err = jsonNum.Int64(); err == nil {
    mField[k] = int(numInt)
    continue
}
mField[k], err = jsonNum.Float64()
```

Back to that `iterateMapFields`, we have to do some recursions if we want iterate all over struct fields, to make sure that we don't miss any `map[string]interface` field
```
switch field.Kind() {
case reflect.Map:
    // execute conversion
case reflect.Struct:
    // recusive loop map fields
    if err = iterateMapFields(field, fn); err != nil {
        return
    }
}
```

There are several more cases, such as `Pointer`, `Slice`, or a map of map, but for my test case, this is good enough.

I put that to test with a struct contain arbitrary field & a struct contain that, to test recursive lookup
```
type foo struct {
	Settings map[string]interface{} `json:"settings"`
}
type bar struct {
	Foo foo `json:settings`
}
```

Test decode to number
```
var sampleFoo = foo{
	Settings: map[string]interface{}{
		"ratio": 0.2,
		"retry": 5,
	},
}
func TestCustomDecodeNumber(t *testing.T) {
	serialized, err := json.MarshalIndent(sampleFoo, " ", " ")
	assert.NoError(t, err)
	assert.Equal(t, string(jsonOut), string(serialized))

	newFoo := foo{}
	assert.NoError(t, decodeUseNumber(serialized, &newFoo))
	assertFoo(t, sampleFoo, newFoo)
}
```

That `assertFoo` just compare sample field type, value with data after run our decoder

```
func assertFoo(t *testing.T, expected, actual foo) {
	assert.NotNil(t, actual.Settings)
	assert.Equal(t, expected.Settings["ratio"], actual.Settings["ratio"])
	assert.Equal(t, expected.Settings["retry"], actual.Settings["retry"])

	fmt.Printf("\n** 'ratio' field value=%v type=%T", actual.Settings["ratio"], actual.Settings["ratio"])
	fmt.Printf("\n** 'retry' field value=%v type=%T \n", actual.Settings["retry"], actual.Settings["retry"])
}
```

There another test to verify nested field works also

Source code: [meomap/jsonexample](https://github.com/meomap/jsonexample/tree/master/jsonnumber)

