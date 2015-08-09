---
title: "Ruby vs. Go 1"
date: "2015-08-09T17:30:00"
description: "The first (slightly unscientific) comparison with Ruby and Go."
categories:
    - "go"
    - "golang"
    - "rubyvsgo"
---

A co-worker built a simple internal tool in Ruby with Sinatra. This tool is essentially a simple front end to a bunch of little simple REST apis. It is a bit slow, but that is okay a ~30 second load time is okay since it is only occasional internal use. I thought to myself looking at the code, with not much more effort you could build a solution in Go / Golang that would work a whole lot better. This isn't the most scientific of comparisons, and you could do a better job with EventMachine - but for now, this is about simulating what the 'real life' ruby solution looks like. The whole project is available on Github here: https://github.com/segfault88/rubyvgo1 so you can follow along.

## The API

I built a simple simulation of the APIs being called. I threw something together in Go, basically it's like this:
* /slow - sleeps for 200-400 ms then returns
* /bad - approximately 50% of the time sleep for 50-100 ms then return, otherwise wait 3 seconds and return
* /timeout - wait 10 seconds then return

The whole thing is simply this:

<pre>
	<code data-language="go">package main

import (
	"fmt"
	"math/rand"
	"net/http"
	"time"

	"github.com/zenazn/goji"
	"github.com/zenazn/goji/web"
)

func slow(c web.C, w http.ResponseWriter, r *http.Request) {
	n := time.Duration(200 + (rand.Int31n(200)))
	time.Sleep(n * time.Millisecond)
	fmt.Fprintf(w, "Done %d ms", n)
}

func bad(c web.C, w http.ResponseWriter, r *http.Request) {
	if rand.Float32() <= 0.5 {
		n := time.Duration(50 + (rand.Int31n(50)))
		time.Sleep(n * time.Millisecond)
		fmt.Fprintf(w, "Done %d ms", n)
	} else {
		time.Sleep(3 * time.Second)
		fmt.Fprint(w, "This api is bad!")
	}
}

func timeout(c web.C, w http.ResponseWriter, r *http.Request) {
	time.Sleep(10 * time.Second)
	fmt.Fprint(w, "That took a while!")
}

func main() {
	rand.Seed(time.Now().UnixNano())
	goji.Get("/slow", slow)
	goji.Get("/bad", bad)
	goji.Get("/timeout", timeout)
	goji.Serve()
}
	</code>
</pre>

## Ruby implementation

The entire Ruby version with Sinatra looks like this:

<pre>
	<code data-language="ruby">require 'rubygems'
require 'sinatra/base'
require 'net/http'

PORT=8000

class MyApp < Sinatra::Base
  get '/' do
    send_file 'index.html'
  end

  get '/slow' do
    t1 = Time.now
    Net::HTTP.get(URI("http://localhost:#{PORT}/slow"))
    "<span class=\"label label-success\">success #{(Time.now - t1).round(2)}</span>"
  end

  get '/bad' do
    t1 = Time.now
    Net::HTTP.get(URI("http://localhost:#{PORT}/bad"))
    t = Time.now - t1
    "<span class=\"label label-#{t > 0.5 ? "danger" : "success"}\">#{t > 0.5 ? "bad" : "good"} #{(t).round(2)}</span>"
  end

  get '/timeout' do
    t1 = Time.now
    Net::HTTP.get(URI("http://localhost:#{PORT}/timeout"))
    "<span class=\"label label-danger\">fail #{(Time.now - t1).round(2)}</span>"
  end
end
	</code>
</pre>

So it looks like this:
![ruby version](/images/rubyvgo1.png)

The first 2 columns load 2-400ms. Column 3 and 4 loads in 50-100ms 50% of the time, 3 seconds otherwise. Column 5 takes 10 seconds.

## Go simple

Go's net/http package is GREAT. We can make a similar system, that actually looks pretty similar to what we do in Ruby. All we need is this:

<pre>
	<code data-language="go">func slow(c web.C, w http.ResponseWriter, r *http.Request) {
	t1 := time.Now()
	_, err := http.Get(fmt.Sprintf("http://localhost:%d/slow", Port))
	if err != nil {
		panic(err) // most basic version, lacking error handling
	}
	fmt.Fprintf(w, "<span class=\"label label-success\">success %s</span>", time.Now().Sub(t1))
}
</code>
</pre>

This is great, it loads a bunch faster. However, the tool works by doing an AJAX request for each 'cell'. Normal brower rules means that only 2 AJAX requests can be outstanding. So it loads a bit faster, and potentially allows more users (the Ruby version only has 2 workers which get blocked when doing the API call), it's not that much faster. Sadface, fail. Let's try another way.

## Go polling

So, let's kick off go-routines immediately on page load, then send updates in batches. Check out the whole file here: https://github.com/segfault88/rubyvgo1/blob/master/go-poll/main.go - this is pretty easy, the interesting part is really this:

<pre>
	<code data-language="go">func callSlow(i int, j int, wg *sync.WaitGroup, r *chan Result) {
	defer wg.Done()
	t1 := time.Now()
	_, err := http.Get(fmt.Sprintf("http://localhost:%d/slow", APIPort))
	panicIfErr(err)
	*r <- Result{i, j, "success", fmt.Sprintf("success %s", time.Now().Sub(t1))}
}
</code></pre>

Here we simply feed the result out on a channel and use a waitgroup to keep track of when the scan is done. Then, when the browser comes along, all we do is package up the updates waiting in the channel and send it along as JSON:

<pre>
	<code data-language="go">response := struct {
		Messages []Result `json:"results"`
		Done     bool     `json:"done"`
	}{
		make([]Result, 0),
		scanner.done,
	}

	// grab all the waiting results
	func() {
		for {
			select {
			case m := <-scanner.results:
				response.Messages = append(response.Messages, m)
			default:
				return
			}
		}
	}()

	// marshal and send json
	json, err := json.Marshal(&response)
	panicIfErr(err)
	w.Header().Set("Content-Type", "application/json")
	w.Write(json)
</code></pre>

This works muuuch faster! 

## Go websocket

Since we're here, lets also try a websocket. Using the "github.com/gorilla/websocket" library makes it a snap to upgrade to a websocket. Again, the code is here: https://github.com/segfault88/rubyvgo1/blob/master/go-websocket/main.go - all you need to do is:

<pre>
	<code data-language="go">var (
	upgrader = websocket.Upgrader{
		ReadBufferSize:  1024,
		WriteBufferSize: 1024,
	}
)

func wsRoute(w http.ResponseWriter, r *http.Request) {
	ws, err := upgrader.Upgrade(w, r, nil)
	if err != nil {
		log.Println(err)
		return
	}

	scanServices(ws)
}
</code>
</pre>

Then you can go ahead and use the websocket to write messages straight to the browser. This is definately the fastest way to go!

<pre>
	<code data-language="go">func callSlow(i int, j int, wg *sync.WaitGroup, r chan Result) {
	defer wg.Done()
	t1 := time.Now()
	_, err := http.Get(fmt.Sprintf("http://localhost:%d/slow", APIPort))
	panicIfErr(err)
	r <- Result{i, j, "success", fmt.Sprintf("success %s", time.Now().Sub(t1))}
}
</code>
</pre>

Now pull from the channel, marshal the JSON and write it to the client:
<pre>
	<code data-language="go">for {
		select {
		case msg := <-messages:
			json, err := json.Marshal(msg)
			panicIfErr(err)
			ws.WriteMessage(websocket.TextMessage, json)
		}
	}
</code>
</pre>

Easy, and really fast.

## Comparison

I did a quick record to a YouTube video to show the relative versions.

<iframe width="420" height="315" src="https://www.youtube.com/embed/YmZ8f6ToUhg" frameborder="0" allowfullscreen></iframe>

Ruby is slow, it's stuck with 2 workers so it blocks a lot. The go version is a fair bit faster, but the limit of 2 outstanding AJAX connections makes it slowish. The polling version is pretty good, and the websocket version is excellent.

## Conclusion

So the original idea for this post is to show that with a **little** more effort, it's easy to get a much better result working in Go. If you look a pure number of lines the go websocket version is something like 3x more than Ruby. However, it works so much faster, will 'scale' and IMO easier to work with in the future.


Questions, comments? Hit me up on twitter or github. Happy hacking!