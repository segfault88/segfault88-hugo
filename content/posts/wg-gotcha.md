---
title: "WaitGroup gotcha"
date: "2015-08-10T00:00:00"
description: "WaitGroup gotcha"
categories:
    - "go"
    - "golang"
---

Quick, what is wrong with this Go code?

<pre>
	<code data-language="go">package main

import (
	"log"
	"math/rand"
	"sync"
	"time"
)

func main() {
	var wg sync.WaitGroup

	for i := 0; i < 100; i++ {
		wg.Add(1)
		go doWork(i, wg)
	}

	log.Println("Starting to wait")
	wg.Wait()
	log.Println("Done")
}

func doWork(i int, wg sync.WaitGroup) {
	defer wg.Done()
	sleepTime := time.Duration(rand.Int31n(10000000))
	time.Sleep(sleepTime)
	log.Printf("%d slept for %d", i, sleepTime)
}
	</code>
</pre>

Answer - the [WaitGroup](http://golang.org/pkg/sync/#WaitGroup) is passed as a value. Internally WaitGroups are structs with a contained integer that gets atomically incremented and decremented. So by passing a value, the internal count gets copied and the copy gets decremented when Done() is called. So this is a deadlock. [Fixed code here](https://github.com/segfault88/misc-go/blob/master/wg-gotcha-fixed.go), simply pass it as a pointer. Helpfully Go will warn you about this:

<pre>
	<code data-language="bash">
$ go vet wg-gotcha.go

wg-gotcha.go:23: doWork passes Lock by value: sync.WaitGroup contains sync.Mutex

exit status 1
	</code>
</pre>

So, moral of the story: Don't pass WaitGroups around by value, and always run [go vet](http://godoc.org/golang.org/x/tools/cmd/vet)!