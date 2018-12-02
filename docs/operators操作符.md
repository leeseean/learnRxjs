## operators操作符

Rxjs中有多达104个operators操作符，操作符的使用都要通过pipe方法。

![avatar](/images/operators.png)

用法示例

```
// RxJS v6+
import { interval } from 'rxjs';
import { bufferCount } from 'rxjs/operators';

// 创建每1秒发出值的 observable
const source = interval(1000);
// 在发出3个值后，将缓冲的值作为数组传递
const bufferThree = source.pipe(bufferCount(3));
// 打印值到控制台
// 输出: [0,1,2]...[3,4,5]
const subscribe = bufferThree.subscribe(val =>
  console.log('Buffered Values:', val)
);

```

源码

每个操作符方法接收一个参数，返回一个函数，以下以map操作符为例

```
//map函数接收两个参数，第一个是函数，第二个是this指针
//返回mapOperation函数，mapOperation函数参数为observable实例
//mapOperation函数执行返回一个处理过的observable实例
export function map<T, R>(project: (value: T, index: number) => R, thisArg?: any): OperatorFunction<T, R> {
  return function mapOperation(source: Observable<T>): Observable<R> {
    if (typeof project !== 'function') {
      throw new TypeError('argument is not a function. Are you looking for `mapTo()`?');
    }
    return source.lift(new MapOperator(project, thisArg));
  };
}

export class MapOperator<T, R> implements Operator<T, R> {
  constructor(private project: (value: T, index: number) => R, private thisArg: any) {
  }

  call(subscriber: Subscriber<R>, source: any): any {
    return source.subscribe(new MapSubscriber(subscriber, this.project, this.thisArg));
  }
}

/**
 * We need this JSDoc comment for affecting ESDoc.
 * @ignore
 * @extends {Ignored}
 */
class MapSubscriber<T, R> extends Subscriber<T> {
  count: number = 0;
  private thisArg: any;

  constructor(destination: Subscriber<R>,
              private project: (value: T, index: number) => R,
              thisArg: any) {
    super(destination);
    this.thisArg = thisArg || this;
  }

  // NOTE: This looks unoptimized, but it's actually purposefully NOT
  // using try/catch optimizations.
  protected _next(value: T) {
    let result: any;
    try {
      result = this.project.call(this.thisArg, value, this.count++);
    } catch (err) {
      this.destination.error(err);
      return;
    }
    this.destination.next(result);
  }
}
```