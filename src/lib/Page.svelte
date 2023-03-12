<script lang="ts">
  // 畫面上的 DOM 物件操作程式
  import * as domUtils from "../utils//dom-utils";
  // 存取 API 資料的程式碼
  import * as dataUtils from "../utils/data-utils";
  import { of, fromEvent, combineLatest, BehaviorSubject, merge } from "rxjs";
  import {
    map,
    switchMap,
    debounceTime,
    distinctUntilChanged,
    share,
    shareReplay,
    filter,
    take,
    startWith,
    catchError,
    mapTo,
    scan,
  } from "rxjs/operators";
  import { onMount } from "svelte";

  onMount(() => {
    const keyword$ = fromEvent(
      document.querySelector("#keyword"),
      "input"
    ).pipe(
      map((event: Event) => (event.target as HTMLInputElement).value), //目的是要取得輸入的值
      startWith(""),
      shareReplay(1)
    );

    // 搭配各種 operators 來查詢建議清單
    // 兼顧效能與資料準確度
    keyword$
      .pipe(
        debounceTime(700),
        //避免重複的查詢：ex本來查rxjs(已完成查詢)之後輸入rxjsbla想了一下又刪掉變回rxjs，這時候就不需要重新查詢
        distinctUntilChanged(),
        //避免內容太少查不出精準的結果
        filter((keyword: string) => keyword.length >= 3),
        switchMap((keyword: string) => dataUtils.getSuggestions(keyword))
      )
      .subscribe((suggestions) => {
        domUtils.fillAutoSuggestions(suggestions);
      });

    const search$ = fromEvent(document.querySelector("#search"), "click");

    // 使用搭配 take(1) 確保只會取得一次
    const keywordForSearch$ = keyword$.pipe(take(1));

    const searchByKeyword$ = search$.pipe(
      switchMap(() => keywordForSearch$),
      filter((keyword) => !!keyword)
    );

    const sortBy$ = new BehaviorSubject({ sort: "stars", order: "desc" });
    const changeSort = (sortField: string) => {
      if (sortField === sortBy$.value.sort) {
        sortBy$.next({
          sort: sortField,
          order: sortBy$.value.order === "asc" ? "desc" : "asc",
        });
      } else {
        sortBy$.next({
          sort: sortField,
          order: "desc",
        });
      }
    };

    fromEvent(document.querySelector("#sort-stars"), "click").subscribe(() => {
      changeSort("stars");
    });
    fromEvent(document.querySelector("#sort-forks"), "click").subscribe(() => {
      changeSort("forks");
    });

    const perPage$ = fromEvent(
      document.querySelector("#per-page"),
      "change"
    ).pipe(map((event: Event) => +(event.target as HTMLSelectElement).value));

    const previousPage$ = fromEvent(
      document.querySelector("#previous-page"),
      "click"
    ).pipe(mapTo(-1));

    const nextPage$ = fromEvent(
      document.querySelector("#next-page"),
      "click"
    ).pipe(mapTo(1));

    const page$ = merge(previousPage$, nextPage$).pipe(
      scan((currentPageIndex, value) => {
        const nextPage = currentPageIndex + +value;
        return nextPage < 1 ? 1 : nextPage;
      }, 1)
    );

    // 顯示頁碼
    page$.subscribe((page) => {
      domUtils.updatePageNumber(page);
    });

    // 顯示 stars 排序資訊
    sortBy$
      .pipe(filter<any>((sort) => sort.sort === "stars"))
      .subscribe((sort) => {
        domUtils.updateStarsSort(sort);
      });

    // 顯示 forks 排序資訊
    sortBy$
      .pipe(filter<any>((sort) => sort.sort === "forks"))
      .subscribe((sort) => {
        domUtils.updateForksSort(sort);
      });

    // 開始進行搜尋的相關條件
    const startSearch$ = combineLatest({
      keyword: searchByKeyword$,
      sort: sortBy$,
      page: page$.pipe(startWith(1)),
      perPage: perPage$.pipe(startWith(10)),
    });

    // 搜尋條件一變更，就執行 domUtils.loading() 遮罩畫面
    startSearch$.subscribe(() => {
      domUtils.loading();
    });

    const getSearchResult = (
      keyword: string,
      sort: string,
      order: string,
      page: number,
      perPage: number
    ) =>
      dataUtils.getSearchResult(keyword, sort, order, page, perPage).pipe(
        map((result) => ({ success: true, message: null, data: result })),
        catchError((error) => {
          return of({
            success: false,
            message: error.response.message,
            data: [],
          });
        })
      );

    // 依照搜尋條件進行搜尋
    const searchResult$ = startSearch$.pipe(
      switchMap(({ keyword, sort, page, perPage }) =>
        getSearchResult(keyword, sort.sort, sort.order, page, perPage)
      ),
      // searchResult$ 有多次訂閱
      // 因此使用 share 避免重複請求資料
      share()
    );

    // 處理畫面顯示
    searchResult$.subscribe((result) => {
      domUtils.fillSearchResult(result.data);
      domUtils.loaded();
    });

    // 處理錯誤提示
    searchResult$
      .pipe(filter<any>((result) => !result.success))
      .subscribe((result) => {
        alert(result.message);
      });
  });
</script>

<div id="searchPanel">
  <h1>Search</h1>
  <div class="form-inline">
    <div class="form-group">
      <input
        type="text"
        id="keyword"
        placeholder="keyword"
        list="suggestions"
        class="form-control"
      />
      <datalist id="suggestions" />
    </div>
    <button id="search" class="btn btn-primary">Search</button>
  </div>
</div>

<h1>Result</h1>

<div
  id="paginationPanel"
  class="d-flex justify-content-start align-items-center"
>
  <select id="per-page" class="form-control" style="width: 100px">
    <option value="10" selected>10</option>
    <option value="20">20</option>
    <option value="50">50</option>
  </select>
  <div class="flex-fill" />
  <button id="previous-page" class="btn btn-primary">Previous Page</button>
  <span id="page-number">1</span>
  <button id="next-page" class="btn btn-primary">Next Page</button>
</div>

<div id="resultPanel">
  <table id="repositories" class="table">
    <thead>
      <th> Name </th>
      <th>
        <span id="sort-stars"> Stars </span>
        <span id="sort-stars-icon" />
      </th>
      <th>
        <span id="sort-forks"> Forks </span>
        <span id="sort-forks-icon" />
      </th>
    </thead>
    <tbody />
  </table>
</div>

<div
  id="loading-block"
  style="display:none; background: rgba(0, 0, 0, 0.6); position: absolute; top: 0; left: 0;"
/>
