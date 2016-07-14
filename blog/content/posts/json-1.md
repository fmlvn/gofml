---
date: "2016-07-14T17:34:19+07:00"
draft: false
title: "JSON encoding pitfalls"
description: "Pitfalls when encode/decode arbitrary data"
tags: [ "golang", "json", "encode", "decode" ]
categories:
  - "Standard Library"
  - "Reflection"
---

### Preface

This is the first post of a series about encoding/decoding JSON in Golang with insight discussion.

### Why JSON ?

JSON is unarguably a prominent data transmission format, especially for web & http requests/ response

But for app configuration, there's no reason to use JSON, absolutely NO. Have a look at these two configurations for same app

JSON
```
{
  "app_name": "test-app",
  "options": {
   "index": 1,
   "interval": 3600000000000,
   "settings": {
    "ratio": 0.2,
    "retry": 5,
    "timeout": 5000000000,
    "url": "http://dummy.com"
   }
  }
 }
```

and YAML
```
app_name: test-app
option:
  index: 1
  interval: 1h
  settings:
    ratio: 0.2
    retry: 5
    timeout: 5s
    url: http://dummy.com
```

All those quotes, and matching brackets are annoying and it took more efforts to write json syntax correctly.

However there's time JSON is the only choice you got then you have to live with that =.= that's my case. But dealing with JSON input data is much much more trouble than we might thought, as you will see soon.


### JSON vs YAML ?
I made a simple script which gives us an overview how JSON & YAML deal with same type of data at different context.

[Sample struct](https://github.com/meomap/jsonexample/blob/master/jsonvsyaml/decode_test.go#L29)
```
var sample = config{
    AppName: "test-app",
    Option: &foo{
        Index:    1,
        Interval: time.Duration(time.Hour),
        Settings: map[string]interface{}{
            "timeout": time.Duration(time.Second * 5),
            "retry":   5,
            "ratio":   0.2,
            "url":     "http://dummy.com",
        },
    },
}
```

So we have a sample struct used for config, this in turn contains another nested setting defined by `foo` struct. This `foo` one is interesting. It hold a `time.Duration` field & a map of string -> `interface{}` so we can basically store anything there.

[foo struct](https://github.com/meomap/jsonexample/blob/master/jsonvsyaml/decode_test.go#L21)
```
type foo struct {
    Index uint64 `json:"index" yaml:"index"`

    Interval time.Duration `json:"interval" yaml:"interval"`

    Settings map[string]interface{} `json:"settings"`
}
```

Let's see how YAML & JSON encode this sample config. The code could be found at
[example](https://github.com/meomap/jsonexample)

```
cd jsonvsyaml
go test go test decode_test.go -v
```

with verbose option, we got this

```
=== RUN   TestYAML
app_name: test-app
option:
  index: 1
  interval: 1h0m0s
  settings:
    ratio: 0.2
    retry: 5
    timeout: 5s
    url: http://dummy.com


** 'index' field value=1 type=uint64
** 'ratio' field value=0.2 type=float64
** 'retry' field value=5 type=int
--- PASS: TestYAML (0.00s)
=== RUN   TestJSON
{
  "app_name": "test-app",
  "options": {
   "index": 1,
   "interval": 3600000000000,
   "settings": {
    "ratio": 0.2,
    "retry": 5,
    "timeout": 5000000000,
    "url": "http://dummy.com"
   }
  }
 }

** 'index' field value=1 type=uint64
** 'ratio' field value=0.2 type=float64
** 'retry' field value=5 type=float64
--- PASS: TestJSON (0.00s)
PASS
ok
```

Notice how they deal with numbers as I noted with `**` during the test run.

Firstly, as we can see, they both can assign parsed value to right field type if we explicitly state field definition. Like the `index` field at `foo` option.

We define
```
Index uint64 `json:"index" yaml:"index"`
```
and got what we wanted at both tests

[yaml](https://github.com/meomap/jsonexample/blob/master/jsonvsyaml/decode_test.go#L83)
[json](https://github.com/meomap/jsonexample/blob/master/jsonvsyaml/decode_test.go#L102)
```
** 'index' field value=1 type=uint64
```

That's good. But for arbitrary number field (actually type definition is `interface{}` so field actual type is determined at runtime when data parsed in by `YAML` or `JSON` lib) we got quite confusing result like the above.

While YAML can assign an input of `5` to field with dynamic type of `int` which is somewhat ideally, parser at JSON assign all number values to `float64` field. That's handling is very annoying, seriously =.=

That JSON package authors know that and they improvised their own type of data to represent number. Guess what, actually, it's just a string named type =)) and developer still have to process that to get what they want

[json.Number](https://golang.org/src/encoding/json/decode.go#L173)

We will see how that `json.Number` and another one `json.RawMessage` come in handy next blog post

Take a look at that serialized json string again and we will find this strange field representation

```
"interval": 3600000000000,
```

Well, that is `time.Duration` type in nano second, so that's how will have to input for an hour setting  :))

for YAML, it's nicely adaptive to human readable form. Too good to deny that
```
interval: 1h
```

And we will see several ways of handling with as you will see at the next posts to come

