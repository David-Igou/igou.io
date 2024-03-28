---
layout: post
title: Simple Go Webhook Receiver
date: 2020-01-19
categories: [golang, webhook, receiver]
---


Go is incredibly powerful and flexible. Something I often find is that I see tutorials that essentially say "Here is how you implement X in Go, Step 1: Download this library I wrote" - It is very rare this is actually necessary. Libraries are still useful to speed up development, but many web based functionalities are very simple to achieve in Go.

```go
package main

import (
	"fmt"
	"log"
	"net/http"
	"encoding/json"
)

func handleWebhook(w http.ResponseWriter, r *http.Request) {
	webhookData := make(map[string]interface{})
	err := json.NewDecoder(r.Body).Decode(&webhookData)
	if err != nil {
		http.Error(w, err.Error(), http.StatusInternalServerError)
		return
	}
	fmt.Println("got webhook payload: ")
	for k, v := range webhookData {
		fmt.Printf("%s : %v\n", k, v)
	}

	switch name := webhookData["username"]; name{
	case "test":
		fmt.Println("Invoked")
	default:
		fmt.Println("Not invoked")
	}
}


func main() {
    log.Println("server started")
	http.HandleFunc("/webhook", handleWebhook)
	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

This example simple takes in the data, spits it out, and has an example switch statement for logic. 
