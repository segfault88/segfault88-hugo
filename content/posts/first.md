---
title: "First post"
date: "2015-08-09"
description: "This is the first post"
categories:
    - "first"
---

First post. Time to resurect this blogging thing!

Lorem ipsum dolor sit amet, consectetur adipiscing elit. Etiam malesuada quam eu justo luctus dapibus. Etiam dictum tellus eu velit mollis imperdiet quis nec nulla. Suspendisse a lectus et lectus hendrerit vulputate lobortis vel sapien. Fusce eleifend nisl eu vestibulum scelerisque. In rutrum lorem magna. Fusce sit amet maximus eros. Ut ullamcorper justo in metus viverra tempus. Aenean placerat sit amet nisi vel viverra. In non pellentesque dui. Nullam auctor vel sapien vehicula mollis. Vestibulum faucibus mollis ipsum vel sodales. Duis eget erat ut purus fermentum malesuada.

<pre>
	<code data-language="python">def openFile(path):
		# syntax highlighting test
	    file = open(path, "r")
	    content = file.read()
	    file.close()
	    return content
    </code>
</pre>

That's easy!

<pre>
	<code data-language="go">
func slow(c web.C, w http.ResponseWriter, r *http.Request) {
	n := time.Duration(200 + (rand.Int31n(200)))
	time.Sleep(n * time.Millisecond)
	fmt.Fprintf(w, "Done %d ms", n)
}
	</code>
</pre>

Noice! Test change 2.