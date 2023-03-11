<script lang="ts">
  import { onMount } from "svelte";
  import { Subject, fromEvent } from "rxjs";
  import { filter } from "rxjs/operators";
  let addBtn: HTMLButtonElement = null;
  let startBtn: HTMLButtonElement = null;
  let errorBtn: HTMLButtonElement = null;
  let completeBtn: HTMLButtonElement = null;

  let countBox: HTMLDivElement = null;
  let statusBox: HTMLDivElement = null;
  let evenBox = (HTMLDivElement = null);

  let counter: number = 0;
  let counter$: Subject<number>;

  onMount(() => {
    fromEvent(startBtn, "click").subscribe(() => {
      counter$ = new Subject();
      counter = 0;
      statusBox.innerHTML = "開始計數";
      // counter$.subscribe((value) => {
      //   countBox.innerHTML = `計數 ${value}`;
      //   // value % 2 === 0 && (evenBox.innerHTML = value);
      // });
      counter$.subscribe({
        next: (value) => {
          countBox.innerHTML = `計數 ${value}`;
        },
        error: msg => statusBox.innerHTML = `錯誤： ${msg}`,
        complete: () => statusBox.innerHTML = "完成",
      })
      counter$.next(counter);
      const evenCounter$ = counter$.pipe(filter(data => data % 2 === 0));
      evenCounter$.subscribe((value)=>{
        evenBox.innerHTML = value;
      })
    });

    fromEvent(addBtn, "click").subscribe(() => {
      counter$.next(++counter);
    })

    fromEvent(errorBtn, 'click').subscribe(() => {
      const reason = prompt('請輸入錯誤訊息')
      counter$.error(reason as string | 'error');
    })

    fromEvent(completeBtn, 'click').subscribe(() => {
      counter$.complete();
    })

  });
</script>

<button bind:this={startBtn} class="reset"> start </button>
<button bind:this={addBtn} class="add"> add </button>
<button bind:this={errorBtn} class="error"> error </button>
<button bind:this={completeBtn} class="complete"> complete </button>

<div bind:this={statusBox}>
  狀態： -
</div>
<div bind:this={countBox}>
  count: {counter}
</div>
<div bind:this={evenBox}>
  even: -
</div>
