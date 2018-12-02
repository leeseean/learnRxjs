## 生成observable的函数

Rxjs中大概提供了32个函数，这些函数执行都返回Observable实例。

![avatar](/images/observable.png)

用法示例

```
// RxJS v6+
import { from } from 'rxjs';

// 将数组作为值的序列发出
const arraySource = from([1, 2, 3, 4, 5]);
// 输出: 1,2,3,4,5
const subscribe = arraySource.subscribe(val => console.log(val));
```

源码

以fromArray为例

```
/**
 * Subscribes to an ArrayLike with a subscriber
 * @param array The array or array-like to subscribe to
 */
export const subscribeToArray = <T>(array: ArrayLike<T>) => (subscriber: Subscriber<T>) => {
  for (let i = 0, len = array.length; i < len && !subscriber.closed; i++) {
    subscriber.next(array[i]);
  }
  if (!subscriber.closed) {
    subscriber.complete();
  }
};

export function fromArray<T>(input: ArrayLike<T>, scheduler?: SchedulerLike) {
  if (!scheduler) {
    return new Observable<T>(subscribeToArray(input));
  } else {
    return new Observable<T>(subscriber => {
      const sub = new Subscription();
      let i = 0;
      sub.add(scheduler.schedule(function () {
        if (i === input.length) {
          subscriber.complete();
          return;
        }
        subscriber.next(input[i++]);
        if (!subscriber.closed) {
          sub.add(this.schedule());
        }
      }));
      return sub;
    });
  }
}
```