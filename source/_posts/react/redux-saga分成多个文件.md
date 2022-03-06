---
title: redux-saga分成多个文件
tags:
  - react
abbrlink: d98c3b78
date: 2019-11-26 15:26:49
---

### 1)

```
// single entry point to start all Sagas at once
export default function* rootSaga() {
  yield [
    saga1(),
    saga2(),
    saga3(),
  ]
}
```

Here the 3 sagas will be run in parallel. The root saga will block until the 3 sagas complete. If one of the 3 fail, the error will be propagated to the root saga which will be killed, which will also kill the other 2 saga

### 2)

```
export default function* root() {
  yield [
    fork(saga1),
    fork(saga2),
    fork(saga3)
  ]
}
```

The only difference I see here is that this time the yield effect will not block because forking is non-blocking, thus the root saga will reach the end but the 3 childs will remain alive. Error behavior is the same as 1)

### 3)

```
export default function* root() {
  yield fork(saga1)
  yield fork(saga2)
  yield fork(saga3)
}
```

I don't see any difference in behavior from 2)

------

# better examples

The problem with forking is that if any of the root saga fails, then the root saga will be killed, and the other sub sagas will also be killed because their parent got killed. In practice this means that your whole app may become unusable (if it relies heavily on sagas) just because of a minor saga error so it's not really good.

### 4)

```
export default function* root() {
  yield spawn(saga1)
  yield spawn(saga2)
  yield spawn(saga3)
}
```

This time, if an error occur in saga1, it will not make root, saga2 and saga3 get killed so only a part of your app stops working in case of error. Somehow this can also be very problematic because the saga1 might be killed due to an error like a failing http request that you didn't catch properly, making the whole feature covered by saga1 unavailable for the app lifetime.

### 5)

[@granmoe](https://github.com/granmoe) has suggested the following way to start sagas in: [#570](https://github.com/redux-saga/redux-saga/issues/570)

```
function* rootSaga () {

  const sagas = [
    saga1,
    saga2,
    saga3,
  ]; 

  yield sagas.map(saga =>
    spawn(function* () {
      while (true) {
        try {
          yield call(saga)
        } catch (e) {
          console.log(e)
        }
      }
    })
  )

}
```

This time, if any of the 3 sagas had an error, it would be automatically restarted. This may, or not, be the desired behavior according to your app.

### 6)

Here's how I handle sagas in my own app:

```
const makeRestartable = (saga) => {
  return function* () {
    yield spawn(function* () {
      while (true) {
        try {
          yield call(saga);
          console.error("unexpected root saga termination. The root sagas are supposed to be sagas that live during the whole app lifetime!",saga);
        } catch (e) {
          console.error("Saga error, the saga will be restarted",e);
        }
        yield delay(1000); // Avoid infinite failures blocking app TODO use backoff retry policy...
      }
    })
  };
};

const rootSagas = [
  domain1saga,
  domain2saga,
  domain3saga,
].map(makeRestartable);

export default function* root() {
  yield rootSagas.map(saga => call(saga));
}
```

I'm using a saga HOC to add error handling to the root sagas. In my app, all root sagas are never supposed to terminate but should block, and if there are errors they should be automatically restarted.

Restarting synchronously can, in my experience, lead to infinite loops (if the saga fails everytime you try to restart it) so I added a hacky delay for now to prevent this issue.

You mentionned different domains in your app so this pattern seems appropriate to your usecase where each domain should somehow have its own root saga.