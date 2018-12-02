## Observable

Observable是一个类，是多个值的推送集合。

## Observable类通过静态方法create创建Observable实例

用法

```
const observable = Observable.create(function (observer) {
  observer.next(1);
  observer.next(2);
  observer.next(3);
  setTimeout(() => {
    observer.next(4);
    observer.complete();
  }, 1000);
});
```

源码

```
  /**
   * 创建一个Observable实例
   * @static true
   * @owner Observable
   * @method create
   * @param {Function} subscribe函数，函数参数为observer对象
   * @return {Observable} 返回一个Observable实例
   **/
  static create: Function = <T>(subscribe?: (subscriber: Subscriber<T>) => TeardownLogic) => {
    return new Observable<T>(subscribe);
  }
```

## Observable实例通过subscribe方法订阅事件，返回Subscriber实例

用法

```
observable.subscribe({
  next: x => console.log('got value ' + x),
  error: err => console.error('something wrong occurred: ' + err),
  complete: () => console.log('done'),
});
observable.subscribe(function (x) {
  console.log(x);
});
```

源码

```
   /**  
   * 参数为observer对象或者next函数，返回Subscriber实例subscription（执行subscription.unsubscribe()订阅取消）
   **/
 subscribe(observerOrNext?: PartialObserver<T> | ((value: T) => void),
            error?: (error: any) => void,
            complete?: () => void): Subscription {

    const { operator } = this;//operator操作对象，包含map，scan等方法
    const sink = toSubscriber(observerOrNext, error, complete);//生成Subscriber实例

    if (operator) {//如果有operator，执行operator，source指Observable实例
      operator.call(sink, this.source);
    } else {//如果没有，订阅Subscriber实例
      sink.add(
        this.source || (config.useDeprecatedSynchronousErrorHandling && !sink.syncErrorThrowable) ?
        this._subscribe(sink) :
        this._trySubscribe(sink)
      );
    }

    if (config.useDeprecatedSynchronousErrorHandling) {
      if (sink.syncErrorThrowable) {
        sink.syncErrorThrowable = false;
        if (sink.syncErrorThrown) {
          throw sink.syncErrorValue;
        }
      }
    }

    return sink;
 }
 _subscribe(subscriber: Subscriber<any>): TeardownLogic {
    const { source } = this;
    return source && source.subscribe(subscriber);//订阅
 }
```

## Observable实例通过forEach方法也可订阅事件，返回一个Promise实例

还有另外一个函式可以达到跟subscribe一样的结果，forEach只接受一个函式，这个函式只负责处理next阶段的行为，且返回的是一个Promise实例，而不是 subscription。返回Promise实例的好处是方便使用await处理异步，例如

```
//输出1,2,3，finish，不加await会先输出finish
async function execute() {
  await Observable.from([1, 2, 3]).delay(1000).forEach(v => console.log(v));
  console.log('finish');
}
```

源码

```
  /**
   * @method forEach
   * @param {Function} next a handler for each value emitted by the observable
   * @param {PromiseConstructor} [promiseCtor] a constructor function used to instantiate the Promise
   * @return {Promise} a promise that either resolves on observable completion or
   *  rejects with the handled error
   */
  forEach(next: (value: T) => void, promiseCtor?: PromiseConstructorLike): Promise<void> {
    promiseCtor = getPromiseCtor(promiseCtor);
    //返回一个Promise实例
    return new promiseCtor<void>((resolve, reject) => {
      // Must be declared in a separate statement to avoid a RefernceError when
      // accessing subscription below in the closure due to Temporal Dead Zone.
      let subscription: Subscription;
      subscription = this.subscribe((value) => {
        try {
          next(value);//执行next方法
        } catch (err) {
          reject(err);
          if (subscription) {
            subscription.unsubscribe();
          }
        }
      }, reject, resolve);
    }) as Promise<void>;
  }
```

## Observable实例通过toPromise将实例转为promise

用法

```
Observable.of('foo').toPromise().then(res => console.log(res));
```

源码

```
  toPromise(promiseCtor?: PromiseConstructorLike): Promise<T> {
    promiseCtor = getPromiseCtor(promiseCtor);
    //执行订阅，并返回一个promise
    return new promiseCtor((resolve, reject) => {
      let value: any;
      this.subscribe((x: T) => value = x, (err: any) => reject(err), () => resolve(value));
    }) as Promise<T>;
  }
```

## Observable实例通过pipe来组合操作符，返回observable实例

用法

```
range(0, 10).pipe(
  filter(x => x % 2 === 0),
  map(x => x + x),
  scan((acc, x) => acc + x, 0)
)
.subscribe(x => console.log(x))
```

源码

```
function pipeFromArray<T, R>(fns: Array<UnaryFunction<T, R>>): UnaryFunction<T, R> {
  if (!fns) {
    return noop as UnaryFunction<any, any>;
  }

  if (fns.length === 1) {
    return fns[0];
  }

  return function piped(input: T): R {// 通过reduce来组合操作符
    return fns.reduce((prev: any, fn: UnaryFunction<T, R>) => fn(prev), input as any);
  };
}
```

```
  pipe(...operations: OperatorFunction<any, any>[]): Observable<any> {
    if (operations.length === 0) {
      return this as any;
    }
    //返回observable实例
    return pipeFromArray(operations)(this);
  }
```
