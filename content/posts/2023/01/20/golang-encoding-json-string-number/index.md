---
title: "[Golang] encoding/jsonで文字列の数値を数値型で扱う"
date: "2023-01-20"
categories: 
  - "プログラミング"
tags: 
  - "golang"
---

以下のようにすることで文字列の数値を数値型で扱うことができます。

```
package main

import (
	"encoding/json"
	"testing"
)

type Test struct {
	Value int `json:"value,string"`
}

func TestUnmarshal(t *testing.T) {
	a := []byte(`{"value": "1000"}`)

	var b Test
	if err := json.Unmarshal(a, &b); err != nil {
		t.Fatal(err)
	}
	if b.Value != 1000 {
		t.Fatal("want: 1000\ngot:", b.Value)
	}
}

func TestMarshal(t *testing.T) {
	want := `{"value":"1000"}`

	got, err := json.Marshal(&Test{Value: 1000})
	if err != nil {
		t.Fatal(err)
	}
	if want != string(got) {
		t.Fatal("want:", want, "\ngot:", string(got))
	}
}
```
