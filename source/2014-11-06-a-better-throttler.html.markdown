---
title: A better throttler?
date: 2014-11-06 00:16 IST
tags: golang, go, rate-limit, throttle, performance, concurrency, improved, mutex
---

In a previous post I wrote a [simple throttler using channels](http://code.websrvr.in/2014/11/04/simple-function-throttler-in-golang/).
On [karlseguin's suggestion](http://www.reddit.com/r/golang/comments/2l6ujc/simple_function_throttler_in_go/clsbvox),
I reimplemented it using `sync.Cond`. [You can check it out at https://bitbucket.org/utils/sync2](https://bitbucket.org/utils/sync2/src/master/throttler.go).

Throttler allows you to restrict a goroutine to a max number of concurrent executions. Think of
it as a mutex which allows 'n' number of locks to be acquired.

~~~go
package sync2

import (
	"sync"
)

//Throttler allows you to restrict a goroutine to a max
//number of concurrent executions. Think of it as a mutex
//which allows 'n' number of locks to be acquired.
type Throttler interface {
	//Lock stops execution and waits if the number of goroutines
	//running has reached the max level of the throttler.
	//It returns the throttler on which it is being called to allow chaining
	//    e.g.
	//    var t= sync2.NewThrottler(10)
	//    func throttledFunc(){
	//      defer t.Lock().Unlock()
	//      ..
	//     }
	Lock() Throttler
	//Unlock releases the throttle on the function/goroutine
	//so other goroutines can continue execution if they were waiting.
	Unlock()
}

type throttler struct {
	l       sync.Locker
	c       *sync.Cond
	running int
	max     int
}

//make sure throttler implements the interface
var _ Throttler = &throttler{}

//NewThrottler creates a throttler which allows you to limit
//the number of concurrent executions of a goroutine or function
func NewThrottler(max int) Throttler {
	l := &sync.Mutex{}
	c := sync.NewCond(l)
	return &throttler{c: c, max: max, l: l}
}

//Unlock releases the throttle on the function/goroutine
//so other goroutines can continue execution if they were waiting.
func (t *throttler) Unlock() {
	t.l.Lock()
	defer t.l.Unlock()
	t.running--
	//signal one of the waiting goroutines to start executing
	t.c.Signal()
}

//Lock stops execution and waits if the number of goroutines
//running has reached the max level of the throttler.
func (t *throttler) Lock() Throttler {
	t.l.Lock()
	defer t.l.Unlock()

	//wait till we have an empty slot
	for (t.max - t.running) < 1 {
		t.c.Wait() //suspends execution of the calling goroutine
	}

	//now we are running
	t.running++
	return t
}
~~~
