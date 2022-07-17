## 함수형 프로그래밍 개념 정리
### 함수형 프로그래밍의 특징
- side Effect가 없다.
- 데이터를 직접 변경하지 않는다.
- 데이터 제어 흐름과 연산을 추상화한다.
- 순수함수를 쓴다.

### 순수함수
- 함수는 하나 이상의 인자를 받는다.
- 반환값이 반드시 존재한다
- 동일한 입력에는 동일한 출력을 보장한다
- 평가 시점에 영향을 받지 않는다.

## debounceTime 코드로 살펴보는 함수형 프로그래밍
```javascript
// debounceTime(500)
const debounceTime500= (source) => {
    const DUE_TIME = 500;
    const scheduler = asyncScheduler;
    const init = (source, subscriber) => {
        let activeTask= null;
        let lastValue= null;
        let lastTime= null;
    
        const emit = () => {
          if (activeTask) {
            // We have a value! Free up memory first, then emit the value.
            activeTask.unsubscribe();
            activeTask = null;
            const value = lastValue;
            lastValue = null;
            subscriber.next(value);
          }
        };
        function emitWhenIdle(this) {
          // This is called `dueTime` after the first value
          // but we might have received new values during this window!
          const targetTime = lastTime + DUE_TIME;
          const now = scheduler.now();
          if (now < targetTime) {
            // On that case, re-schedule to the new target
            activeTask = this.schedule(undefined, targetTime - now);
            subscriber.add(activeTask);
            return;
          }
    
          emit();
        }
    
        source.subscribe(
          createOperatorSubscriber(
            subscriber,
            (value) => {
              lastValue = value;
              lastTime = scheduler.now();
    
              // Only set up a task if it's not already up
              if (!activeTask) {
                activeTask = scheduler.schedule(emitWhenIdle, DUE_TIME);
                subscriber.add(activeTask);
              }
            },
            () => {
              // Source completed.
              // Emit any pending debounced values then complete
              emit();
              subscriber.complete();
            },
            // Pass all errors through to consumer.
            undefined,
            () => {
              // Finalization.
              lastValue = activeTask = null;
            }
          )
        );
    }
    if (hasLift(source)) {
      return source.lift(function (this, liftedSource) {
        try {
          return init(liftedSource, this);
        } catch (err) {
          this.error(err);
        }
      });
    }
    throw new TypeError('Unable to lift unknown Observable type');
  }
```
- debounceTime에 첫번째 인자에 500을 입력하면 다음과 같은 함수가 리턴된다.
- source를 인자로 받고, 이 인자가 lift 메서드를 가진 경우에 source.lift에 debounceTime이 적용된 함수를 인자로 넘긴다.
- debounceTime 세부 구현에서는 4가지의 변수를 받아야한다.
- dueTime, scheduler, source, subscriber이다.

```typescript
import { asyncScheduler } from '../scheduler/async';
import { Subscription } from '../Subscription';
import { MonoTypeOperatorFunction, SchedulerAction, SchedulerLike } from '../types';
import { operate } from '../util/lift';
import { createOperatorSubscriber } from './OperatorSubscriber';

/**
 * Emits a notification from the source Observable only after a particular time span
 * has passed without another source emission.
 *
 * <span class="informal">It's like {@link delay}, but passes only the most
 * recent notification from each burst of emissions.</span>
 *
 * ![](debounceTime.png)
 *
 * `debounceTime` delays notifications emitted by the source Observable, but drops
 * previous pending delayed emissions if a new notification arrives on the source
 * Observable. This operator keeps track of the most recent notification from the
 * source Observable, and emits that only when `dueTime` has passed
 * without any other notification appearing on the source Observable. If a new value
 * appears before `dueTime` silence occurs, the previous notification will be dropped
 * and will not be emitted and a new `dueTime` is scheduled.
 * If the completing event happens during `dueTime` the last cached notification
 * is emitted before the completion event is forwarded to the output observable.
 * If the error event happens during `dueTime` or after it only the error event is
 * forwarded to the output observable. The cache notification is not emitted in this case.
 *
 * This is a rate-limiting operator, because it is impossible for more than one
 * notification to be emitted in any time window of duration `dueTime`, but it is also
 * a delay-like operator since output emissions do not occur at the same time as
 * they did on the source Observable. Optionally takes a {@link SchedulerLike} for
 * managing timers.
 *
 * ## Example
 *
 * Emit the most recent click after a burst of clicks
 *
 * ```ts
 * import { fromEvent, debounceTime } from 'rxjs';
 *
 * const clicks = fromEvent(document, 'click');
 * const result = clicks.pipe(debounceTime(1000));
 * result.subscribe(x => console.log(x));
 * ```
 *
 * @see {@link audit}
 * @see {@link auditTime}
 * @see {@link debounce}
 * @see {@link sample}
 * @see {@link sampleTime}
 * @see {@link throttle}
 * @see {@link throttleTime}
 *
 * @param {number} dueTime The timeout duration in milliseconds (or the time
 * unit determined internally by the optional `scheduler`) for the window of
 * time required to wait for emission silence before emitting the most recent
 * source value.
 * @param {SchedulerLike} [scheduler=async] The {@link SchedulerLike} to use for
 * managing the timers that handle the timeout for each value.
 * @return A function that returns an Observable that delays the emissions of
 * the source Observable by the specified `dueTime`, and may drop some values
 * if they occur too frequently.
 */
export function debounceTime<T>(dueTime: number, scheduler: SchedulerLike = asyncScheduler): MonoTypeOperatorFunction<T> {
  return operate((source, subscriber) => {
    let activeTask: Subscription | null = null;
    let lastValue: T | null = null;
    let lastTime: number | null = null;

    const emit = () => {
      if (activeTask) {
        // We have a value! Free up memory first, then emit the value.
        activeTask.unsubscribe();
        activeTask = null;
        const value = lastValue!;
        lastValue = null;
        subscriber.next(value);
      }
    };
    function emitWhenIdle(this: SchedulerAction<unknown>) {
      // This is called `dueTime` after the first value
      // but we might have received new values during this window!

      const targetTime = lastTime! + dueTime;
      const now = scheduler.now();
      if (now < targetTime) {
        // On that case, re-schedule to the new target
        activeTask = this.schedule(undefined, targetTime - now);
        subscriber.add(activeTask);
        return;
      }

      emit();
    }

    source.subscribe(
      createOperatorSubscriber(
        subscriber,
        (value: T) => {
          lastValue = value;
          lastTime = scheduler.now();

          // Only set up a task if it's not already up
          if (!activeTask) {
            activeTask = scheduler.schedule(emitWhenIdle, dueTime);
            subscriber.add(activeTask);
          }
        },
        () => {
          // Source completed.
          // Emit any pending debounced values then complete
          emit();
          subscriber.complete();
        },
        // Pass all errors through to consumer.
        undefined,
        () => {
          // Finalization.
          lastValue = activeTask = null;
        }
      )
    );
  });
}
```
- debounceTime.ts파일을 살펴보면 dueTime, scheduler는 함수를 직접 호출할 때 고정되고, source와 subscriber는 operator함수에서 제공한다.
- debounceTime 함수에서는 현재 ActiveTask가 있는지 여부, lastValue와 lastTime만을 평가한다.

```typescript
export function operate<T, R>(
  init: (liftedSource: Observable<T>, subscriber: Subscriber<R>) => (() => void) | void
): OperatorFunction<T, R> {
  return (source: Observable<T>) => {
    if (hasLift(source)) {
      return source.lift(function (this: Subscriber<R>, liftedSource: Observable<T>) {
        try {
          return init(liftedSource, this);
        } catch (err) {
          this.error(err);
        }
      });
    }
    throw new TypeError('Unable to lift unknown Observable type');
  };
}
```
- operator함수는 함수를 반환합니다.
- souce에 lift함수가 있는지 확인하고, tryCatch 예외처리를 담당합니다.
