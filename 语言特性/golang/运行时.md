# GO语言Runtime解析

## Go语言Runtime的启动流程
1. 调用osinit函数
2. 调用schedinit函数
3. 调用mstart函数

## GMP
GMP模型是Go用来管理goroutine的方式，其中有三个概念，G是goroutine代表一个需要调度的任务，P代表一个goroutine的处理器，是M和G的中间层，提供M需要的上下文环境，调度G到M上去执行，M对应一个系统线程是真正工作的线程。他们之间的关系是M对应一个P，P中有一个G的队列，P会取出队列中的G，把G放到M上去执行。

### 关键变量
* **idlepMask和timerpMask**：这两个是用来看哪个p空闲哪个p有定时器。
* allp：所有的proccesser队列。
* allg：所有的goroutine队列。
* sched：
* g0：持有调度栈的goroutine。深度参与运行时调度，包括goroutine创建、大内存分配和CGO函数执行。

## P

### P的创建
P通过procresize函数进行初始化，P的初始化除了在调用schedinit函数中被调用，还在startTheWorldWithSema函数中被调用，这个函数和gc有关，涉及的gc阶段有gcStart，gcMarkDone，gcMarkTermination，startTheWorld。做这个操作的时候会获取sched.lock锁。


这个函数具体做了以下一些事：
* 根据gomaxprocs的大小维护allp切片大小，两个大小保持一致。同时也会维护idlepMask和timerpMask。
* 创建新的P，并且进行P的初始化
* 绑定m0线程和allp[0]
* 释放不再使用的P
* 如果allp大了，截断allp
* 设置除allp[0]以外的P设置为idle，加入到全局空闲队列

## G

### G的创建
创建一个goroutine时，会调用newproc函数创建G，将这个G放到当前正在运行的G所属的P的队列上面。创建G是通过newproc1函数来进行的，创建出G后通过runqput函数放到队列中，可能是全局队列也可能是P的本地队列。

#### newproc1获取G流程
1. 调用gfget函数，获取一个G。
    * gfget函数会看curP和sched上面有空闲G的话，就从curP或者sched的gFree上拿空闲的G。
    * 如果curP没有的话，就从sched的gFree上拿空闲的G放到curP上，直到curP有了32个空闲的G。
    * curP中有了空闲的G了，就pop出一个空闲的G返回。
2. 没拿到就调用**malg(_StackMin)**创建goroutine。
    * new一个G结构体出来，为了防止被GC回收，这个G的状态设为Gdead，然后扔到allg队列上去。
    * 给G分配stack，把函数参数拷贝到这个G的栈上
3. 最终拿到一个G后，调用memmove()把fn函数的参数拷贝到栈上。
4. 设置G的一些参数，sp，pc等，把状态从Gdead改为Grunnable。
总结创建流程就是，G的创建流程是首先从当前的P上拿空闲G，没有就看sched上有的话从sched上面拿空闲G，sched上面也没有的话，就创建一个新的G，然后把fn的变量拷贝到G的栈上，再去设置G的参数和sp，pc，sched调度信息等等，最后G的状态变为runnable。

**备注**
1. 在newproc1的步骤4，会将G.sched.pc指向goexit()+PCQuantum，**应该是用于在goroutine被调度完成后执行的，还不确定**。
2. goexit()是所有goroutine调用栈的栈顶，最后都会返回到这个函数里面。


#### runqput放入队列流程
通过调用newproc1拿到一个G之后呢这个G就是一个Grunnable状态的了，我们通过runqput把G放到队列上等待被P调度。流程如下：
1. 通过newproc来创建的G在调用runqput的时候next=true，会放在P的next上面执行。
2. 如果next上面有其他G的话，得把这个oldG扔到P的runq上去，如果runq满了，把这个oldG加上P的runq上的一批G，扔到全局队列里面去。
总结一下就是，如果next=true，就会把G扔到P的runnext中，如果runnext有oldG，这个oldG会走next=false的逻辑。next=false的逻辑是把G扔到队列中，可能是P的本地队列，也可能是全局队列，由P的runq是否有空位来控制。

#### G创建过程总结
从上面的逻辑来总结一下创建goroutine的一个流程，首先通过go关键字创建goroutine会调用newproc函数，这个函数有两个主要逻辑，第一个逻辑是创建一个G，通过newproc1函数来获得一个G，这个G可能是从P的gFree或者sched的gFree中获取的，P和sched中是否有空余的gFree，优先拿P的，P中没有拿sched的。如果P和sched中都没有空闲的G，就会调用malg创建一个G。获取到一个G后需要把fn的参数替换到G的sp上，替换完成后会设置G的参数，sp，pc，status等。第二个逻辑是把拿到的G放到队列上，因为newproc调用runqput的时候next=true，所以会替换P上的老runnext，把旧的runnext扔到本地队列或全局队列里面等待调度。



### G的调度
G的调度是在schedule函数中执行的。在执行完schedinit函数进行初始化之后会执行mstart函数，这个函数中会调用到schedule函数，调用顺序为mstart -> mstart0 -> mstart1 -> schedule，schedule函数是一个不会退出的大循环，主要的作用就是查找runnable的G然后运行它。所以要解析G的调度，就要从这个函数下手。
#### schedule执行流程
1. 先机率取全局队列的G
2. 没取全局取本地P的runq中的G
3. 本地没有再调findrunnable拿一个G
    * 从本地队列找
    * 从全局队列找
    * 从网络轮询器找
    * 调runqsteal去其他P那里偷
4. 拿到了就调execute执行，谁都没有G就阻塞等待
    * 做一些工作，比如建立M和G的关系，改G的状态，改栈地址等等
    * 调用gogo函数把goroutine放到当前线程上执行
    * 执行完后进入goexit，做一些清除工作，然后把goroutine加入到P的gFree，最后重新调用schedule
总结一下，G的调度是在schedule中进行的，shedule函数会找到一个runnable的G，然后执行它，查找的方式是首先有几率的从全局拿，没从全局拿或者全局没有，再去本地runq中拿，本地没有调用findrunnable函数，这个函数会先从本地拿，再从全局拿，再从netpoll拿，最后去其他P里面偷。

#### 触发schedule的场景
1. 主动挂起：gopark -> park_m -> schedule
2. 系统调用：exitsyscall -> exitsyscall0 -> schedule
3. 协作式调度：Gosched -> gosched_m -> goschedImpl -> schedule
4. 系统监控：sysmon -> retake -> preemptone -> schedule
5. 协程退出：goexit -> goexit0 -> schedule

## M
### M的创建
运行时通过startm()来启动一个线程，该函数通过mge()来从sched.midle中获取M，如果没有的话通过newm()创建一个M并且和P进行绑定然后运行M，newm()中调用allocm()来创建一个M结构体并且和P绑定，然后调用newosproc来创建系统线程，并将M和系统线程进行绑定。


## 结构源码
```go 
type g struct {
	// Stack parameters.
	// stack describes the actual stack memory: [stack.lo, stack.hi).
	// stackguard0 is the stack pointer compared in the Go stack growth prologue.
	// It is stack.lo+StackGuard normally, but can be StackPreempt to trigger a preemption.
	// stackguard1 is the stack pointer compared in the C stack growth prologue.
	// It is stack.lo+StackGuard on g0 and gsignal stacks.
	// It is ~0 on other goroutine stacks, to trigger a call to morestackc (and crash).
	stack       stack   // offset known to runtime/cgo
	stackguard0 uintptr // offset known to liblink
	stackguard1 uintptr // offset known to liblink

	_panic    *_panic // innermost panic - offset known to liblink
	_defer    *_defer // innermost defer
	m         *m      // current m; offset known to arm liblink
	sched     gobuf
	syscallsp uintptr // if status==Gsyscall, syscallsp = sched.sp to use during gc
	syscallpc uintptr // if status==Gsyscall, syscallpc = sched.pc to use during gc
	stktopsp  uintptr // expected sp at top of stack, to check in traceback
	// param is a generic pointer parameter field used to pass
	// values in particular contexts where other storage for the
	// parameter would be difficult to find. It is currently used
	// in three ways:
	// 1. When a channel operation wakes up a blocked goroutine, it sets param to
	//    point to the sudog of the completed blocking operation.
	// 2. By gcAssistAlloc1 to signal back to its caller that the goroutine completed
	//    the GC cycle. It is unsafe to do so in any other way, because the goroutine's
	//    stack may have moved in the meantime.
	// 3. By debugCallWrap to pass parameters to a new goroutine because allocating a
	//    closure in the runtime is forbidden.
	param        unsafe.Pointer
	atomicstatus uint32
	stackLock    uint32 // sigprof/scang lock; TODO: fold in to atomicstatus
	goid         int64
	schedlink    guintptr
	waitsince    int64      // approx time when the g become blocked
	waitreason   waitReason // if status==Gwaiting

	preempt       bool // preemption signal, duplicates stackguard0 = stackpreempt
	preemptStop   bool // transition to _Gpreempted on preemption; otherwise, just deschedule
	preemptShrink bool // shrink stack at synchronous safe point

	// asyncSafePoint is set if g is stopped at an asynchronous
	// safe point. This means there are frames on the stack
	// without precise pointer information.
	asyncSafePoint bool

	paniconfault bool // panic (instead of crash) on unexpected fault address
	gcscandone   bool // g has scanned stack; protected by _Gscan bit in status
	throwsplit   bool // must not split stack
	// activeStackChans indicates that there are unlocked channels
	// pointing into this goroutine's stack. If true, stack
	// copying needs to acquire channel locks to protect these
	// areas of the stack.
	activeStackChans bool
	// parkingOnChan indicates that the goroutine is about to
	// park on a chansend or chanrecv. Used to signal an unsafe point
	// for stack shrinking. It's a boolean value, but is updated atomically.
	parkingOnChan uint8

	raceignore     int8     // ignore race detection events
	sysblocktraced bool     // StartTrace has emitted EvGoInSyscall about this goroutine
	tracking       bool     // whether we're tracking this G for sched latency statistics
	trackingSeq    uint8    // used to decide whether to track this G
	runnableStamp  int64    // timestamp of when the G last became runnable, only used when tracking
	runnableTime   int64    // the amount of time spent runnable, cleared when running, only used when tracking
	sysexitticks   int64    // cputicks when syscall has returned (for tracing)
	traceseq       uint64   // trace event sequencer
	tracelastp     puintptr // last P emitted an event for this goroutine
	lockedm        muintptr
	sig            uint32
	writebuf       []byte
	sigcode0       uintptr
	sigcode1       uintptr
	sigpc          uintptr
	gopc           uintptr         // pc of go statement that created this goroutine
	ancestors      *[]ancestorInfo // ancestor information goroutine(s) that created this goroutine (only used if debug.tracebackancestors)
	startpc        uintptr         // pc of goroutine function
	racectx        uintptr
	waiting        *sudog         // sudog structures this g is waiting on (that have a valid elem ptr); in lock order
	cgoCtxt        []uintptr      // cgo traceback context
	labels         unsafe.Pointer // profiler labels
	timer          *timer         // cached timer for time.Sleep
	selectDone     uint32         // are we participating in a select and did someone win the race?

	// Per-G GC state

	// gcAssistBytes is this G's GC assist credit in terms of
	// bytes allocated. If this is positive, then the G has credit
	// to allocate gcAssistBytes bytes without assisting. If this
	// is negative, then the G must correct this by performing
	// scan work. We track this in bytes to make it fast to update
	// and check for debt in the malloc hot path. The assist ratio
	// determines how this corresponds to scan work debt.
	gcAssistBytes int64
}

type m struct {
	g0      *g     // goroutine with scheduling stack
	morebuf gobuf  // gobuf arg to morestack
	divmod  uint32 // div/mod denominator for arm - known to liblink

	// Fields not known to debuggers.
	procid        uint64            // for debuggers, but offset not hard-coded
	gsignal       *g                // signal-handling g
	goSigStack    gsignalStack      // Go-allocated signal handling stack
	sigmask       sigset            // storage for saved signal mask
	tls           [tlsSlots]uintptr // thread-local storage (for x86 extern register)
	mstartfn      func()
	curg          *g       // current running goroutine
	caughtsig     guintptr // goroutine running during fatal signal
	p             puintptr // attached p for executing go code (nil if not executing go code)
	nextp         puintptr
	oldp          puintptr // the p that was attached before executing a syscall
	id            int64
	mallocing     int32
	throwing      int32
	preemptoff    string // if != "", keep curg running on this m
	locks         int32
	dying         int32
	profilehz     int32
	spinning      bool // m is out of work and is actively looking for work
	blocked       bool // m is blocked on a note
	newSigstack   bool // minit on C thread called sigaltstack
	printlock     int8
	incgo         bool   // m is executing a cgo call
	freeWait      uint32 // if == 0, safe to free g0 and delete m (atomic)
	fastrand      [2]uint32
	needextram    bool
	traceback     uint8
	ncgocall      uint64      // number of cgo calls in total
	ncgo          int32       // number of cgo calls currently in progress
	cgoCallersUse uint32      // if non-zero, cgoCallers in use temporarily
	cgoCallers    *cgoCallers // cgo traceback if crashing in cgo call
	doesPark      bool        // non-P running threads: sysmon and newmHandoff never use .park
	park          note
	alllink       *m // on allm
	schedlink     muintptr
	lockedg       guintptr
	createstack   [32]uintptr // stack that created this thread.
	lockedExt     uint32      // tracking for external LockOSThread
	lockedInt     uint32      // tracking for internal lockOSThread
	nextwaitm     muintptr    // next m waiting for lock
	waitunlockf   func(*g, unsafe.Pointer) bool
	waitlock      unsafe.Pointer
	waittraceev   byte
	waittraceskip int
	startingtrace bool
	syscalltick   uint32
	freelink      *m // on sched.freem

	// mFixup is used to synchronize OS related m state
	// (credentials etc) use mutex to access. To avoid deadlocks
	// an atomic.Load() of used being zero in mDoFixupFn()
	// guarantees fn is nil.
	mFixup struct {
		lock mutex
		used uint32
		fn   func(bool) bool
	}

	// these are here because they are too large to be on the stack
	// of low-level NOSPLIT functions.
	libcall   libcall
	libcallpc uintptr // for cpu profiler
	libcallsp uintptr
	libcallg  guintptr
	syscall   libcall // stores syscall parameters on windows

	vdsoSP uintptr // SP for traceback while in VDSO call (0 if not in call)
	vdsoPC uintptr // PC for traceback while in VDSO call

	// preemptGen counts the number of completed preemption
	// signals. This is used to detect when a preemption is
	// requested, but fails. Accessed atomically.
	preemptGen uint32

	// Whether this is a pending preemption signal on this M.
	// Accessed atomically.
	signalPending uint32

	dlogPerM

	mOS

	// Up to 10 locks held by this m, maintained by the lock ranking code.
	locksHeldLen int
	locksHeld    [10]heldLockInfo
}

type p struct {
	id          int32
	status      uint32 // one of pidle/prunning/...
	link        puintptr
	schedtick   uint32     // incremented on every scheduler call
	syscalltick uint32     // incremented on every system call
	sysmontick  sysmontick // last tick observed by sysmon
	m           muintptr   // back-link to associated m (nil if idle)
	mcache      *mcache
	pcache      pageCache
	raceprocctx uintptr

	deferpool    [5][]*_defer // pool of available defer structs of different sizes (see panic.go)
	deferpoolbuf [5][32]*_defer

	// Cache of goroutine ids, amortizes accesses to runtime·sched.goidgen.
	goidcache    uint64
	goidcacheend uint64

	// Queue of runnable goroutines. Accessed without lock.
	runqhead uint32
	runqtail uint32
	runq     [256]guintptr
	// runnext, if non-nil, is a runnable G that was ready'd by
	// the current G and should be run next instead of what's in
	// runq if there's time remaining in the running G's time
	// slice. It will inherit the time left in the current time
	// slice. If a set of goroutines is locked in a
	// communicate-and-wait pattern, this schedules that set as a
	// unit and eliminates the (potentially large) scheduling
	// latency that otherwise arises from adding the ready'd
	// goroutines to the end of the run queue.
	//
	// Note that while other P's may atomically CAS this to zero,
	// only the owner P can CAS it to a valid G.
	runnext guintptr

	// Available G's (status == Gdead)
	gFree struct {
		gList
		n int32
	}

	sudogcache []*sudog
	sudogbuf   [128]*sudog

	// Cache of mspan objects from the heap.
	mspancache struct {
		// We need an explicit length here because this field is used
		// in allocation codepaths where write barriers are not allowed,
		// and eliminating the write barrier/keeping it eliminated from
		// slice updates is tricky, moreso than just managing the length
		// ourselves.
		len int
		buf [128]*mspan
	}

	tracebuf traceBufPtr

	// traceSweep indicates the sweep events should be traced.
	// This is used to defer the sweep start event until a span
	// has actually been swept.
	traceSweep bool
	// traceSwept and traceReclaimed track the number of bytes
	// swept and reclaimed by sweeping in the current sweep loop.
	traceSwept, traceReclaimed uintptr

	palloc persistentAlloc // per-P to avoid mutex

	_ uint32 // Alignment for atomic fields below

	// The when field of the first entry on the timer heap.
	// This is updated using atomic functions.
	// This is 0 if the timer heap is empty.
	timer0When uint64

	// The earliest known nextwhen field of a timer with
	// timerModifiedEarlier status. Because the timer may have been
	// modified again, there need not be any timer with this value.
	// This is updated using atomic functions.
	// This is 0 if there are no timerModifiedEarlier timers.
	timerModifiedEarliest uint64

	// Per-P GC state
	gcAssistTime         int64 // Nanoseconds in assistAlloc
	gcFractionalMarkTime int64 // Nanoseconds in fractional mark worker (atomic)

	// gcMarkWorkerMode is the mode for the next mark worker to run in.
	// That is, this is used to communicate with the worker goroutine
	// selected for immediate execution by
	// gcController.findRunnableGCWorker. When scheduling other goroutines,
	// this field must be set to gcMarkWorkerNotWorker.
	gcMarkWorkerMode gcMarkWorkerMode
	// gcMarkWorkerStartTime is the nanotime() at which the most recent
	// mark worker started.
	gcMarkWorkerStartTime int64

	// gcw is this P's GC work buffer cache. The work buffer is
	// filled by write barriers, drained by mutator assists, and
	// disposed on certain GC state transitions.
	gcw gcWork

	// wbBuf is this P's GC write barrier buffer.
	//
	// TODO: Consider caching this in the running G.
	wbBuf wbBuf

	runSafePointFn uint32 // if 1, run sched.safePointFn at next safe point

	// statsSeq is a counter indicating whether this P is currently
	// writing any stats. Its value is even when not, odd when it is.
	statsSeq uint32

	// Lock for timers. We normally access the timers while running
	// on this P, but the scheduler can also do it from a different P.
	timersLock mutex

	// Actions to take at some time. This is used to implement the
	// standard library's time package.
	// Must hold timersLock to access.
	timers []*timer

	// Number of timers in P's heap.
	// Modified using atomic instructions.
	numTimers uint32

	// Number of timerDeleted timers in P's heap.
	// Modified using atomic instructions.
	deletedTimers uint32

	// Race context used while executing timer functions.
	timerRaceCtx uintptr

	// preempt is set to indicate that this P should be enter the
	// scheduler ASAP (regardless of what G is running on it).
	preempt bool

	// Padding is no longer needed. False sharing is now not a worry because p is large enough
	// that its size class is an integer multiple of the cache line size (for any of our architectures).
}
```