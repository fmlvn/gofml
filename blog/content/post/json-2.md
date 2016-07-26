---
date: "2016-07-23T14:46:01+07:00"
draft: true
title: "JSON decoding numbers"
description: "Decode arbitrary JSON data contain number"
tags: [ "golang", "json", "encode", "decode", "interface"]
categories:
  - "Standard Library"
  - "Reflection"
---

### Preface

At previous post, we had encountered a problem when decode arbitray JSON data containing number. To recap, we decode JSON data into a `map[string]interface{}` and suffered type change, from `int` to `float64`, see more at ["json-1.md"].

This issue was referenced at another post too, please see [http://devs.cloudimmunity.com/gotchas-and-common-mistakes-in-go-golang/index.html#json_num]

That blog author proposed these options:
1. Using the float value as-is
2. Convert the float value to the integer type you need.
3. Using a Decoder type to unmarshal JSON using `json.Number`.
4. Using a `struct` type that maps your numeric value to the numeric type you need.
5. Using a `struct` type that maps your number value to `json.RawMessage`

IMO, the best way to handling that should always the *option 4* which to not decode arbitrary data at all. We should know struct we are decoding to. There's other benifits of that besiding handling number. We can use JSON tag to vaidate input data, so there's no more overhead to check type, check nil, empty value after that. [https://golang.org/pkg/encoding/json/#Marshal]

The worst must be the *option 5* as it will break code resusability. With struct field type `json.RawMessage` we emilate other encoding/decoding type such as `xml`, `yaml`

In case we have no choice but have to decode to a `map[string]interface{}`, say it depends on other developer usage then we can consider both `option 2 & 3`. We will use decode with to `json.Number` type then convert its string value to the actual number type. For generic usage, we will use `reflection` to inspect number runtime type.
