## Observable

Observable是一个类，是多个值的推送集合。

Observable是一个类，包含静态方法create，protoType上有subscribe，

## Observable类通过静态方法create创建Observable实例

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

源码

```
   /**  
   * 参数为observer对象或者next函数，返回Subscriber实例subscription（执行subscription()订阅取消）
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
