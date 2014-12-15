scala-future-vs-rxscala
=======================
Scala Future & Promise to RxScala cheat sheet

This is a wiki page to note some important points when convert code from (using) [Scala Future & Promise](http://docs.scala-lang.org/overviews/core/futures.html) to [RxScala](https://github.com/ReactiveX/RxScala).

Please create a PR if you...

### Future ~ Observable, but:

1. Future hold 1 result or hold an Error, Observable hold 0, 1, some, infinite result or hold an Error

  ```scala
  //a Future[T] hold 0 or 1 underlying value of type T:
  val f1 = Future successful 5 //T is Int, f1 hold 1 value = 5
  val fE = Future failed new Exception("foo") //T is Nothing, fE hold an Exception
  
  //an Observable[T] hold 0, some, or infinite underlying value of type T:
  val o1 = Observable just 5 //T is Int, o1 hold 1 value = 5
  val o3 = Observable from Seq(1,3,5) //o3 hold 3 value
  val oInf = o3.repeat //oInf hold infinite value: 1,3,5,1,3,5,...
  //T is Nothing, o0 hold 0 value - invokes onCompleted on observers
  val o0 = Observable.empty
  //T is Nothing, oE hold an Exception - invokes onError on observers
  val oE = Observable error new Exception("foo")
  ```

2. Future can "hold" > 1 values: Use Future[Seq[T]]

  ```scala
  def f(i: Int): Future[Long] = Future successful i.toLong
  val f3: Future[Seq[Long]] = Future.traverse(Seq(1,3,5))(f)
  ```

3. So, when code using Observable, we MUST care about whether the obs is similar to Future[T] or Future[Seq[T]]

4. Observable.flatMat, concatMap, toSeq

  To preserve order of the underlying values, we MUST use concatMap instead of flatMap:
  
  ```scala
  def delay(x: Int): Observable[Int] = Observable.timer(x.seconds).map(_ => x)
  val o2 = Observable.just(3, 1)
  o2.flatMap(delay).foreach(print) //print: 13
  o2.concatMap(delay).foreach(print) //print: 31
  o2.concatMap(delay).toSeq.foreach(print) //print: Buffer(3, 1)
  ```

5. aFuture1.onFailure: Unit vs anObservable.doOnError == anObservable

  ```scala
  f onFailure { case e => logger.error(e) } //expression has type Unit
  o doOnError { e => logger.error(e) } //expression's result is same as o
  ```

6. Future.andThen vs Observable.doOnXXX

  ```scala  
  f andThen { case Failure(e) => logger.error(e) } //f.andThen's result is same as f
  //is similar to:
  o doOnError { e => logger.error(e) } //o.doOnXXX's result is same as o
  
  f andThen { case Success(v) => ??? }
  //is similar to: (ONLY if o hold an Error or hold exactly 1 value)
  o doOnNext { v => ??? }
  ```

7. Future.onComplete is DIFERENT from Observable.doOnCompleted

  Future.onComplete(f) alway run f while Observable.doOnCompleted only run f if the observable NOT hold an Error.
  ```scala
  Future failed new Exception onComplete {
    case Failure(e) => print("fail")
    case Success(_) =>
  }
  
  Observable error new Exception doOnCompleted {
    print("fail") //never go here
  }
  
  aFuture onComplete { _ => ??? }
  // is similar to
  anObservable doOnTerminate { ??? }
  // or
  anObservable finallyDo { ??? }
  ```

8. Observable can be [`hot` or `cold`](https://github.com/ReactiveX/RxJava/wiki/Observable#hot-and-cold-observables). Future is alway `hot`

  ```scala
  val obs = Observable[Int] { observer =>
    print("This will never be printed!")
    observer.onNext(0)
    observer.onCompleted()
  }
  obs.map(_ + 1)
    .doOnNext(println) //not print
    .finallyDo(println("This will never be printed!!"))
  
  val f = Future { print("This will BE printed before `f` is created!"); 0 }
  f.map(_ + 1)
    .andThen { case Success(i) => println(i) } //print 1
    .onComplete(_ => println("This will BE printed!"))
  ```
