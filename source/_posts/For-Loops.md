---
title: For-loops and goroutines
date: 2018-03-18
banner:
  url: /images/For-Loop.png
  width: 640
  height: 360
tags:
    - Go
categories: GoNotes
---
Some interesting things occur when you use goroutines inside for-loops in go.



What happens when you want to concurrently run a number of goroutines using a
for-loop. Well, let's try it out.

```go
type serverConfig struct {
  endpoint string
  path     string
}

func helloHandler(w http.ResponseWriter, r *http.Request) {
  fmt.Fprintf(w, "Hi there, I am %s!", r.URL.Path[1:])
}

func main() {
  servers := []serverConfig{
    {endpoint: ":8000", path: "hello1"},
    {endpoint: ":8001", path: "hello2"},
    {endpoint: ":8002", path: "hello3"},
  }

  c := make(chan os.Signal)
  signal.Notify(c, os.Interrupt, os.Kill)

  for _, server := range servers {
    go func() {
      s := http.NewServeMux()
      s.HandleFunc(fmt.Sprintf("/%s", server.path), helloHandler)
      log.Printf("running server at %s", server.endpoint)
      log.Fatal(http.ListenAndServe(server.endpoint, s))
    }()
  }
  <-c
}
```

Running this in a go environment yields this error.


```bash
2018/03/18 16:16:24 running server at :8002
2018/03/18 16:16:24 running server at :8002
2018/03/18 16:16:24 running server at :8002
2018/03/18 16:16:24 listen tcp :8002: bind: address already in use

Process finished with exit code 1
```

What the! it seems like when the loop runs, it uses the same `:8002` value
over and over again. Why is that?

It turns out, this happens because of golang's memory referencing between goroutines
according to Stephen Grider in his Golang Udemy course (I highly recommend you watch
if you are new to go). When you run a for-loop in go, the `range servers` assigns a value to
`server` in the main goroutine. When we reference `server` in the child goroutine in the
inner loop, we are essentially sharing the memory address between the main goroutine and
the child goroutine.

If memory addresses are shared between goroutines, there is a potential to run into
data race conditions especially if the value associated with the shared memory address is
changed repeatedly. It turns out that in my case, the main goroutine runs through the loop
much faster than the child goroutines. By the time the child goroutines are executed, they
all read the last value that was populated into the `server` shared memory address in the
loop. Hence we see each server trying to start on the same port. How do we fix this? Well,
we can take advantage of go's pass-by-value feature when dealing with functions.

Let's see what that looks like:

```go
...

for _, server := range servers {
    go func(srv serverConfig) {
      s := http.NewServeMux()
      s.HandleFunc(fmt.Sprintf("/%s", srv.path), helloHandler)
      log.Printf("running server at %s", srv.endpoint)
      log.Fatal(http.ListenAndServe(srv.endpoint, s))
    }(server)
  }

...

```

The code was shortened for brevity, but the change is simple and small. All we have done
is change the function literal to accept a `serverConfig` we are essentially giving each
child routine its own copy a `serverConfig`.

This is the output we get after running our code.
```bash
2018/03/18 17:14:56 running server at :8002
2018/03/18 17:14:56 running server at :8000
2018/03/18 17:14:56 running server at :8001
```
Yay! no exit errors, and each server is started on its own port that we configured!
