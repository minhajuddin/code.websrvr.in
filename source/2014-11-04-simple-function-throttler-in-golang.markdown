---
title: Simple function throttler in golang
date: 2014-11-04
tags:
---

Go is fun, I love writing code in go. I am rewriting the whole backend for
http://www.websrvr.in/ using go and am loving every moment of it. Here is a
small throttler I wrote to throttle goroutines in my code. The simplicity of
this blows my mind, In another language I would have to bend over backwards to
get this done.


[Play with it in Go playground](http://play.golang.org/p/WjtUP59Xps)
~~~golang

package main

import (
	"fmt"
	"sync"
	"time"
)

//start of throttler
type Throttler chan bool

func NewThrottler(n int) Throttler {
	return make(chan bool, n)
}

func (t Throttler) Fill() Throttler {
	t <- true
	fmt.Print("+") //for demo
	return t
}

func (t Throttler) Drain() {
	<-t
	fmt.Print("-") //for demo
}

//end of throttler

func main() {
	//creates a throttler which runs a maximum of 2 routines concurrently
	t := NewThrottler(2)

	//demo code
	st := time.Now()
	wg := &sync.WaitGroup{}
	for i := 0; i < 10; i++ {
		wg.Add(1)
		go func() {
			//used to throttle
			defer wg.Done()
			defer t.Fill().Drain()
			fmt.Print("W")
			time.Sleep(1 * time.Millisecond) //to simulate work
		}()
	}

	wg.Wait()
	fmt.Println("\nTime to finish:", time.Now().Sub(st))
}

//Output
//+W+W--+W+W--+W+W--+W+W--+W+W--
//Time to finish: 5.756916ms

~~~
