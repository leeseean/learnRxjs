## Subject

Subject既是Observable又是Observer。
是Observable是因为它继承于Observable。
是Observer是因为它也有next、error、complete方法，可用作subscribe的参数。
不同于Observable，它可以向多个Observer多路推送数值。普通的Observable并不具备多路推送的能力（每一个Observer都有自己独立的执行环境），而Subject可以共享一个执行环境。

用法

```
import { Subject, from } from 'rxjs';

const subject = new Subject<number>();
//作为observable订阅
subject.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
subject.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});
//作为observe执行next方法
subject.next(1);
subject.next(2);
//作为observer当做subscribe
from([1, 2, 3]).subscribe(subject); 的参数
```

源码

```
//继承了Observable，所以它也是Observable
class Subject<T> extends Observable<T> implements SubscriptionLike {

  [rxSubscriberSymbol]() {
    return new SubjectSubscriber(this);
  }

  observers: Observer<T>[] = [];//观察对象是一个数组，可以多播

  closed = false;

  isStopped = false;

  hasError = false;

  thrownError: any = null;

  constructor() {
    super();
  }

  /**@nocollapse
   * @deprecated use new Subject() instead
  */
  static create: Function = <T>(destination: Observer<T>, source: Observable<T>): AnonymousSubject<T> => {
    return new AnonymousSubject<T>(destination, source);
  }

  lift<R>(operator: Operator<T, R>): Observable<R> {
    const subject = new AnonymousSubject(this, this);
    subject.operator = <any>operator;
    return <any>subject;
  }
  //observer的next方法
  next(value?: T) {
    if (this.closed) {
      throw new ObjectUnsubscribedError();
    }
    if (!this.isStopped) {
      const { observers } = this;
      const len = observers.length;
      const copy = observers.slice();
      for (let i = 0; i < len; i++) {
        copy[i].next(value);
      }
    }
  }
  //observer的error方法
  error(err: any) {
    if (this.closed) {
      throw new ObjectUnsubscribedError();
    }
    this.hasError = true;
    this.thrownError = err;
    this.isStopped = true;
    const { observers } = this;
    const len = observers.length;
    const copy = observers.slice();
    for (let i = 0; i < len; i++) {
      copy[i].error(err);
    }
    this.observers.length = 0;
  }
  //observer的complete方法
  complete() {
    if (this.closed) {
      throw new ObjectUnsubscribedError();
    }
    this.isStopped = true;
    const { observers } = this;
    const len = observers.length;
    const copy = observers.slice();
    for (let i = 0; i < len; i++) {
      copy[i].complete();
    }
    this.observers.length = 0;
  }

  unsubscribe() {
    this.isStopped = true;
    this.closed = true;
    this.observers = null;
  }

  /** @deprecated This is an internal implementation detail, do not use. */
  _trySubscribe(subscriber: Subscriber<T>): TeardownLogic {
    if (this.closed) {
      throw new ObjectUnsubscribedError();
    } else {
      return super._trySubscribe(subscriber);
    }
  }

  /** @deprecated This is an internal implementation detail, do not use. */
  _subscribe(subscriber: Subscriber<T>): Subscription {
    if (this.closed) {
      throw new ObjectUnsubscribedError();
    } else if (this.hasError) {
      subscriber.error(this.thrownError);
      return Subscription.EMPTY;
    } else if (this.isStopped) {
      subscriber.complete();
      return Subscription.EMPTY;
    } else {
      this.observers.push(subscriber);//把订阅对象推入observers数组，实现多个观察
      return new SubjectSubscription(this, subscriber);
    }
  }

  /**
   * Creates a new Observable with this Subject as the source. You can do this
   * to create customize Observer-side logic of the Subject and conceal it from
   * code that uses the Observable.
   * @return {Observable} Observable that the Subject casts to
   */
  asObservable(): Observable<T> {
    const observable = new Observable<T>();
    (<any>observable).source = this;
    return observable;
  }
}
```

## 借助multicast操作符实现多播

用法

```
import { from, Subject } from 'rxjs';
import { multicast } from 'rxjs/operators';

const source = from([1, 2, 3]);
const subject = new Subject();
const multicasted = source.pipe(multicast(subject));

// 通过`subject.subscribe({...})`订阅Subject的Observer
multicasted.subscribe({
  next: (v) => console.log(`observerA: ${v}`)
});
multicasted.subscribe({
  next: (v) => console.log(`observerB: ${v}`)
});

// 让Subject从数据源订阅开始生效
multicasted.connect();
```

multicast 实现之中，它包含如下代码

```
let subjectFactory: () => Subject<T>;
if (typeof subjectOrSubjectFactory === 'function') {
    subjectFactory = <() => Subject<T>>subjectOrSubjectFactory;
} else {
    subjectFactory = function subjectFactory() {
    return <Subject<T>>subjectOrSubjectFactory;
    };
}

if (typeof selector === 'function') {
    return source.lift(new MulticastOperator(subjectFactory, selector));
}

const connectable: any = Object.create(source, connectableObservableDescriptor);
connectable.source = source;
connectable.subjectFactory = subjectFactory;

return <ConnectableObservable<R>> connectable;
```
只有在不传入 selector 函数的情况下，multicast 才返回 ConnectableObservable。如果传入 selector 函数的话，会使用 lift 机制来使得源 observable 创建出适当类型的 observable 。不需要在返回的 observable 上调用 connect方法，并且在 selector 函数的作用域中会共享源 observable 。

ConnectableObservable的实现中，含如下代码

```
  connect(): Subscription {
    let connection = this._connection;
    if (!connection) {
      this._isComplete = false;
      connection = this._connection = new Subscription();
      //add加入执行队列
      connection.add(this.source
        .subscribe(new ConnectableSubscriber(this.getSubject(), this)));
      if (connection.closed) {
        this._connection = null;
        connection = Subscription.EMPTY;
      } else {
        this._connection = connection;
      }
    }
    return connection;
  }
```

由上可见connect()方法决定Observable何时开始执行。由于调用connect()后，Observable开始执行，返回一个Subscription供调用者来终止执行。