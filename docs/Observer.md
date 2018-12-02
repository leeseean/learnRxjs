## Observer

Observer是一个对象，用来做subscribe方法的参数，这个对象含有一个closed属性，以及next，error，complete三个方法

用法

```
const observer = {
  next: x => console.log('got value ' + x),
  error: err => console.error('something wrong occurred: ' + err),
  complete: () => console.log('done'),
};
observable.subscribe(observer); 
```

源码

```
export const empty: Observer<any> = {
  closed: true,
  next(value: any): void { /* noop */},
  error(err: any): void {
    if (config.useDeprecatedSynchronousErrorHandling) {
      throw err;
    } else {
      hostReportError(err);
    }
  },
  complete(): void { /*noop*/ }
};
```