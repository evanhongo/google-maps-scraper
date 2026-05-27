# CLI 模式深度解說

本文以「外層流程 → 核心引擎 → 業務細節」三層深入解說 `google-maps-scraper` 的 CLI 模式（`RunModeFile`）。

---

## 一、模式定義：什麼時候進入 CLI 模式？

CLI 模式（`RunModeFile`）的觸發條件在 `runner/runner.go:214-229` 的隱式 switch：

```go
switch {
case cfg.AwsLambdaInvoker:  ...
case cfg.AwsLamdbaRunner:   ...
case cfg.WebRunner || (cfg.Dsn == "" && cfg.InputFile == ""):
    cfg.RunMode = RunModeWeb         // 沒有任何輸入 → 預設開 web UI
case cfg.Dsn == "":
    cfg.RunMode = RunModeFile        // ← 有 -input 但沒 -dsn ＝ CLI 模式
case cfg.ProduceOnly:
    cfg.RunMode = RunModeDatabaseProduce
case cfg.Dsn != "":
    cfg.RunMode = RunModeDatabase
}
```

**關鍵設計**：沒有 `--mode=cli` 這種旗標，模式是由「有沒有 input file」「有沒有 dsn」「-web 是不是被指定」隱式推斷。`-input queries.txt` 而沒 `-dsn` 就鎖定 CLI 模式。

入口在 `main.go:67-84`，`runnerFactory` 把 `cfg` 交給 `filerunner.New(cfg)`，回傳 `runner.Runner` 介面（只有 `Run(ctx)` 與 `Close(ctx)` 兩個方法）。

---

## 二、初始化：filerunner.New 的三件事

`filerunner.New` 是個三步驟的 builder（`filerunner.go:32-54`）：

```go
ans := &fileRunner{cfg: cfg}
ans.setInput()    // 1. 把 -input 變成 io.Reader
ans.setWriters()  // 2. 決定結果寫到哪
ans.setApp()      // 3. 配置 scrapemate 引擎
```

### 2.1 `setInput()` — 字串 "stdin" 是哨兵

```go
switch r.cfg.InputFile {
case "stdin":
    r.input = os.Stdin
default:
    f, err := os.Open(r.cfg.InputFile)
    r.input = f
}
```

字串 `"stdin"` 作為哨兵值，允許 pipeline：

```bash
echo "coffee in Berlin" | ./google-maps-scraper -input stdin -results out.csv
```

### 2.2 `setWriters()` — 四選一的輸出策略

依優先序判定（`filerunner.go:173-218`）：

| 條件 | Writer | 行為 |
|---|---|---|
| `-writer dir:Sym` | `runner.LoadCustomWriter` | `plugin.Open` 動態載入 `.so`/`.dll` |
| `-leadsdb-api-key` | `leadsdb.New(apiKey)` | 批次 100 筆 / 60 秒上傳 LeadsDB |
| `-json` | `jsonwriter.NewJSONWriter` | NDJSON 串流 |
| 預設 | `csvwriter.NewCsvWriter` | CSV，可寫 `os.Stdout` 或檔案 |

注意三者是「**互斥**」的（switch-case）—— 同時指定 plugin 與 LeadsDB 時，**plugin 勝出**，這是排序決定的細節。`-results stdout` 同樣是哨兵字串，預設輸出到終端機。

### 2.3 `setApp()` — scrapemate 引擎配置

這是 CLI 模式最關鍵的配置點（`filerunner.go:220-267`）：

```go
opts := []func(*scrapemateapp.Config) error{
    scrapemateapp.WithConcurrency(r.cfg.Concurrency),         // 並行 worker 數
    scrapemateapp.WithExitOnInactivity(r.cfg.ExitOnInactivityDuration),  // 閒置超時
}

if len(r.cfg.Proxies) > 0 {
    opts = append(opts, scrapemateapp.WithProxies(r.cfg.Proxies))
}

if !r.cfg.FastMode {
    // 正常路徑：Playwright + Chromium，DisableImages 省頻寬
    if r.cfg.Debug {
        opts = append(opts, scrapemateapp.WithJS(scrapemateapp.Headfull(), scrapemateapp.DisableImages()))
    } else {
        opts = append(opts, scrapemateapp.WithJS(scrapemateapp.DisableImages()))
    }
} else {
    // Fast mode 路徑：純 HTTP，stealth Firefox fingerprint
    opts = append(opts, scrapemateapp.WithStealth("firefox"))
}

if !r.cfg.DisablePageReuse {
    opts = append(opts,
        scrapemateapp.WithPageReuseLimit(2),    // 看似多餘
        scrapemateapp.WithPageReuseLimit(200),  // 但 scrapemate 接受重複，後者覆蓋前者
    )
}
```

**`WithPageReuseLimit(2)` 後面接 `WithPageReuseLimit(200)` 不是 bug**：scrapemate option 函式可以重複呼叫，後者勝出。這是 200 個 Chromium Page 的暖池上限。

---

## 三、Run() 主迴圈

`Run(ctx)` 是這整套 CLI 模式的指揮中心（`filerunner.go:56-137`）：

```go
func (r *fileRunner) Run(ctx context.Context) (err error) {
    var seedJobs []scrapemate.IJob
    t0 := time.Now().UTC()

    // [A] 延遲執行的 telemetry 上報
    defer func() {
        elapsed := time.Now().UTC().Sub(t0)
        params := map[string]any{
            "job_count": len(seedJobs),
            "duration":  elapsed.String(),
        }
        if err != nil { params["error"] = err.Error() }
        evt := tlmt.NewEvent("file_runner", params)
        _ = runner.Telemetry().Send(ctx, evt)
    }()

    // [B] 建立執行範圍的單例
    dedup := deduper.New()
    exitMonitor := exiter.New()

    // [C] 路徑分岐：grid 模式或一般模式
    if r.cfg.GridBBox != "" {
        if r.cfg.FastMode { return fmt.Errorf("-fast-mode cannot be used together with -grid-bbox") }
        bbox, _ := grid.ParseBoundingBox(r.cfg.GridBBox)
        seedJobs, err = runner.CreateGridSeedJobs(...)
    } else {
        seedJobs, err = runner.CreateSeedJobs(...)
    }

    // [D] 設定 exit monitor
    exitMonitor.SetSeedCount(len(seedJobs))
    ctx, cancel := context.WithCancel(ctx)
    defer cancel()
    exitMonitor.SetCancelFunc(cancel)
    go exitMonitor.Run(ctx)   // 背景 goroutine 等 doneCh

    // [E] 啟動 scrapemate，會阻塞到所有 job 完成或 ctx 被 cancel
    err = r.app.Start(ctx, seedJobs...)
    return err
}
```

幾個值得注意的決策：

1. **`dedup` 與 `exitMonitor` 在 Run 內建立**：意味著每次 `Run` 都是乾淨狀態，可以多次呼叫 `runner.Runner.Run(ctx)`（雖然實務上只跑一次）。
2. **`SetSeedCount(len(seedJobs))` 在 `app.Start` 之前**：必須先告訴 exiter 有多少種子任務，否則 exiter 永遠覺得「還沒收齊」。
3. **`go exitMonitor.Run(ctx)`**：背景 goroutine 阻塞在 `doneCh`，當所有計數歸位才 cancel 上層 context，導致 `app.Start` 自然返回。
4. **互斥檢查在 Run，不在 ParseConfig**：`-fast-mode` + `-grid-bbox` 的衝突是執行期才報錯。

---

## 四、種子任務的構造

### 4.1 `parseQueryLine` — `#!#` 自訂 ID 約定

`runner/jobs.go:245-265` 定義了輸入格式：

```go
if before, after, ok := strings.Cut(line, "#!#"); ok {
    q.text = strings.TrimSpace(before)
    q.id   = strings.TrimSpace(after)
}
```

範例：

```
Matsuhisa Athens
coffee in Berlin#!#my-berlin-job
dentist in Tokyo
```

- 第 1、3 行：沒有 `#!#`，會分配 UUID 當 ID。
- 第 2 行：自訂 ID `my-berlin-job`，會傳播到該行衍生出的**所有 Entry 的 `input_id` 欄位**。

**為什麼這麼設計？** 因為 Google Maps 一個 query 會展開出 ~120 個 Entry，CLI 使用者需要追溯每筆資料是哪個原始 query 的產物。

### 4.2 `CreateSeedJobs` — 標準路徑

對每個 query line，依 `fastmode` 決定建 `GmapJob` 或 `SearchJob`（`runner/jobs.go:21-133`）：

```go
if !fastmode {
    job = gmaps.NewGmapJob(id, langCode, query, maxDepth, email, geoCoordinates, zoom, opts...)
} else {
    jparams := gmaps.MapSearchParams{
        Location: gmaps.MapLocation{Lat: lat, Lon: lon, ZoomLvl: float64(zoom), Radius: radius},
        Query:    query, Hl: langCode,
        ViewportW: 1920, ViewportH: 450,
    }
    job = gmaps.NewSearchJob(&jparams, opts...)
}
```

**Fast mode 要求座標**：因為它是純 HTTP 模擬 Google 內部 endpoint，沒有座標就無法構造 `pb` 參數。Normal mode 則可選座標（影響搜尋中心）。

### 4.3 `CreateGridSeedJobs` — Cartesian product

Grid 模式做 `queries × cells` 的 cartesian product（`runner/jobs.go:141-214`）：

```go
for _, q := range queries {
    for _, cell := range cells {
        cellID := uuid.New().String()
        if queryID != "" { cellID = fmt.Sprintf("%s-%s", queryID, cellID) }
        job := gmaps.NewGmapJob(cellID, langCode, queryText, maxDepth, email,
            cell.GeoCoordinates(), zoom, opts...)
        jobs = append(jobs, job)
    }
}
```

每個 cell 都被當成一次獨立搜尋的中心點。**鄰居 cell 結果重複完全靠共享的 `dedup` 抑制** —— 注意所有 GmapJob 都拿到同一個 `dedup` 指標，這是 grid 模式能正確去重的關鍵。

---

## 五、GmapJob —— 搜尋結果頁的爬取

### 5.1 URL 構造（`gmaps/job.go:34-80`）

```go
if geoCoordinates != "" && zoom > 0 {
    mapURL = fmt.Sprintf("https://www.google.com/maps/search/%s/@%s,%dz", query, ...)
} else {
    mapURL = fmt.Sprintf("https://www.google.com/maps/search/%s", query)
}
```

兩種 URL 格式：

- 有座標：`https://www.google.com/maps/search/coffee/@52.5200,13.4050,15z` — Google 會以此為中心搜尋。
- 無座標：`https://www.google.com/maps/search/coffee` — Google 用 IP 推斷地點。

**`maxRetries: 3`**、**`Priority: PriorityLow`**（讓 PlaceJob 優先消耗）、**`URLParams: {"hl": langCode}`**（語系）。

### 5.2 `BrowserActions` —— 瀏覽器互動的完整序列（`gmaps/job.go:185-257`）

```go
func (j *GmapJob) BrowserActions(ctx context.Context, page scrapemate.BrowserPage) scrapemate.Response {
    pageResponse, err := page.Goto(j.GetFullURL(), scrapemate.WaitUntilDOMContentLoaded)
    clickRejectCookiesIfRequired(page)                    // [1] 自動拒絕 cookie
    _ = page.WaitForURL(page.URL(), defaultTimeout)        // [2] 等可能的 redirect

    // [3] 檢測「只有一個結果」的情況
    err = page.WaitForSelector(`div[role='feed']`, 10*time.Second)
    var singlePlace bool
    if err != nil {
        waitCtx, _ := context.WithTimeout(ctx, time.Second*5)
        singlePlace = waitUntilURLContains(waitCtx, page, "/maps/place/")
    }

    if singlePlace {
        // 直接取頁面內容當回應
        resp.URL = page.URL()
        body, _ := page.Content()
        resp.Body = []byte(body)
        return resp
    }

    // [4] 一般情況：滾動觸發 lazy-load
    _, err = scroll(ctx, page, j.MaxDepth, `div[role='feed']`)
    body, _ := page.Content()
    resp.Body = []byte(body)
    return resp
}
```

**`clickRejectCookiesIfRequired`** 用 JavaScript 同步搜尋並點擊（比一次次 Playwright locator 快），支援英文 "reject"/"decline" 與德文 "ablehnen"：

```js
const consentForm = document.querySelector('form[action*="consent.google"]');
if (consentForm) { ... }
const buttons = document.querySelectorAll('button, input[type="submit"]');
for (const btn of buttons) {
    const text = (btn.textContent || btn.value || '').toLowerCase();
    if (text.includes('reject') || text.includes('decline') || text.includes('ablehnen')) {
        btn.click(); return true;
    }
}
```

**單一結果重定向處理**：如果輸入是非常精確的 query（如 "Matsuhisa Athens"），Google 會繞過列表頁直接跳到 place 頁面。`waitUntilURLContains` 用 150ms 間隔輪詢 URL，5 秒內看到 `/maps/place/` 就把整個 HTML body 當回應送出。

### 5.3 滾動算法（`gmaps/job.go:300-373`）

```go
waitTime := 100.
cnt := 0
const timeout = 500
const maxWait2 = 2000

for i := 0; i < maxDepth; i++ {
    cnt++
    waitTime2 := timeout * cnt           // JS 內 setTimeout 等待，遞增
    if waitTime2 > timeout { waitTime2 = maxWait2 }

    scrollHeight, err := page.Eval(fmt.Sprintf(expr, waitTime2))  // 注入 JS 滾動
    // ... 把 scrollHeight 轉成 int (可能是 int 或 float64)

    if height == currentScrollHeight { break }    // 高度沒變 → 到底了
    currentScrollHeight = height

    waitTime *= 1.5                       // Go 層 page.WaitForTimeout，1.5x 遞增
    if waitTime > maxWait2 { waitTime = maxWait2 }
    page.WaitForTimeout(time.Duration(waitTime) * time.Millisecond)
}
```

**兩層等待**：

1. **JS 層**：`el.scrollTop = el.scrollHeight; setTimeout(resolve, waitTime2)` —— 等 Google 把新項目 DOM 渲染完。
2. **Go 層**：`page.WaitForTimeout(...)` —— 等網路請求穩定下來才再滾。

**1.5 倍指數退避** 是為慢網路與走代理的情境設計，最高 capped 在 2000ms。**短路停止條件**：兩次滾動後高度相同就停（已到底）。

### 5.4 `Process` —— 把 HTML 變成 PlaceJob 列表

兩種輸出路徑（`gmaps/job.go:114-183`）：

**路徑 1：單一結果頁面**（URL 已被 redirect 到 `/maps/place/...`）：

```go
if strings.Contains(resp.URL, "/maps/place/") {
    placeJob := NewPlaceJob(j.ID, j.LangCode, resp.URL, j.ExtractEmail, j.ExtractExtraReviews, jopts...)
    next = append(next, placeJob)
}
```

**路徑 2：列表頁面**，用 goquery 抽連結：

```go
doc.Find(`div[role=feed] div[jsaction]>a`).Each(func(_ int, s *goquery.Selection) {
    if href := s.AttrOr("href", ""); href != "" {
        nextJob := NewPlaceJob(j.ID, j.LangCode, href, j.ExtractEmail, j.ExtractExtraReviews, jopts...)
        if j.Deduper == nil || j.Deduper.AddIfNotExists(ctx, href) {  // 共享 deduper 過濾
            next = append(next, nextJob)
        }
    }
})
```

**`div[role=feed] div[jsaction]>a`** 是寫死的 CSS selector，Google 一改前端版本就會壞 —— 這是這類 scraper 的固有脆弱性。

之後通報 exiter：

```go
j.ExitMonitor.IncrPlacesFound(len(next))   // 「我找到 N 個地方」
j.ExitMonitor.IncrSeedCompleted(1)         // 「我這個 seed 完成了」
```

**`GmapJob.UseInResults()` 回傳 `false`**：搜尋頁本身不產出 Entry，只生子任務。

---

## 六、PlaceJob —— 單一商家詳情頁

### 6.1 從 `APP_INITIALIZATION_STATE` 取 JSON（`gmaps/place.go:297-319`）

```javascript
(function() {
    if (!window.APP_INITIALIZATION_STATE || !window.APP_INITIALIZATION_STATE[3]) return null;
    const appState = window.APP_INITIALIZATION_STATE[3];
    for (const key of Object.keys(appState)) {
        const arr = appState[key];
        if (Array.isArray(arr)) {
            for (const idx of [6, 5]) {                       // 先試 index 6，再試 5
                const item = arr[idx];
                if (typeof item === 'string' && item.startsWith(")]}'")) {  // Google 防 eval 標記
                    return item;
                }
            }
        }
    }
    return null;
})()
```

Google 把 place 資料藏在這個 hydration array 的固定位置。`)]}'` 是 Google 業界用了多年的 **JSON-prefix-hijacking 防護標記**（防止舊瀏覽器把回應當 JS eval）—— 我們找到這個前綴的字串就知道是要的資料。

### 6.2 兩層重試：`getRaw` × `extractJSON`（`gmaps/place.go:206-282`）

```go
// 內層：每 200ms 重試一次，直到拿到非 nil 非空字串
func (j *PlaceJob) getRaw(ctx context.Context, page scrapemate.BrowserPage) (any, error) {
    for {
        select {
        case <-ctx.Done(): return nil, fmt.Errorf("timeout: %w", ctx.Err())
        default:
            raw, err := page.Eval(js)
            if err != nil || raw == nil { time.Sleep(200 * time.Millisecond); continue }
            if str, ok := raw.(string); ok && str == "" { time.Sleep(200 * time.Millisecond); continue }
            return raw, nil
        }
    }
}

// 外層：最多 2 次嘗試，每次給 30 秒，失敗就 Reload 整頁
func (j *PlaceJob) extractJSON(page scrapemate.BrowserPage) ([]byte, error) {
    const maxRetries = 2
    for attempt := range maxRetries {
        ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
        rawI, err := j.getRaw(ctx, page)
        cancel()
        if err != nil {
            if attempt < maxRetries-1 {
                if reloadErr := page.Reload(scrapemate.WaitUntilDOMContentLoaded); reloadErr == nil {
                    continue
                }
            }
            return nil, err
        }
        // ... 處理 raw
        raw = strings.TrimSpace(strings.TrimPrefix(raw, `)]}'`))
        return []byte(raw), nil
    }
}
```

**兩層的意義**：

- 內層處理「JS hydration 還沒跑完」—— 200ms 間隔輪詢直到 `APP_INITIALIZATION_STATE` 出現。
- 外層處理「頁面就是壞掉了」—— 整頁 reload 再試一次。

### 6.3 Process 的三條岔路（`gmaps/place.go:72-144`）

```go
entry, err := EntryFromJSON(raw)
entry.ID = j.ParentID  // ★ 把 GmapJob 的 ID 傳下來，這就是 input_id 欄位

// 處理 extra reviews（如果有的話）
if reviews, ok := resp.Meta["reviews_raw"].(FetchReviewsResponse); ok { entry.AddExtraReviews(...) }
if domReviews, ok := resp.Meta["dom_reviews"].([]DOMReview); ok { entry.UserReviewsExtended = append(...) }

// 三條岔路
if j.ExtractEmail && entry.IsWebsiteValidForEmail() {
    // 岔路 A：開 EmailExtractJob 抓 email
    emailJob := NewEmailJob(j.ID, &entry, opts...)
    j.UsageInResultststs = false               // ★★ 關鍵：告訴 scrapemate「不要寫我」
    return nil, []scrapemate.IJob{emailJob}, nil
} else if j.ExitMonitor != nil && !j.WriterManagedCompletion {
    // 岔路 B：不抓 email，直接寫，立刻記 IncrPlacesCompleted
    j.ExitMonitor.IncrPlacesCompleted(1)
}

return &entry, nil, err                        // 岔路 C：交給 writer
```

**`j.UsageInResultststs = false` 是個很巧妙的責任轉移**（拼字有 typo 但保留）：

- scrapemate 跑完 `Process` 後會檢查 `j.UseInResults()`，如果為 `true` 就把第一個回傳值寫到 writers。
- 當我們 spawn 子 job 接力處理同一個 entry，必須關掉自己的寫入旗標，否則 entry 會被寫兩次（一次沒 email，一次有 email）。

### 6.4 `IsWebsiteValidForEmail` 的過濾邏輯（`entry.go:127-145`）

```go
func (e *Entry) IsWebsiteValidForEmail() bool {
    if e.WebSite == "" { return false }
    needles := []string{"facebook", "instragram", "twitter"}  // ★ "instragram" 拼錯
    for i := range needles {
        if strings.Contains(e.WebSite, needles[i]) { return false }
    }
    return true
}
```

**業務邏輯**：商家的「網站」欄位常常是 Facebook page、Instagram profile 或 Twitter handle，這些頁面要嘛是 SPA（regex 抓不到 email）、要嘛刻意混淆 email、要嘛 IP 會被封 —— 直接跳過。**注意 `instragram` 有拼錯**，意味著 Instagram URL 不會被排除（這是個小 bug）。

---

## 七、EmailExtractJob —— 最佳努力的 email 抓取

### 7.1 URL 標準化（`emailjob.go:150-176`）

```go
func normalizeGoogleURL(rawURL string) string {
    if strings.HasPrefix(rawURL, "/url?q=") {
        fullURL := "https://www.google.com" + rawURL
        parsed, _ := url.Parse(fullURL)
        if target := parsed.Query().Get("q"); target != "" {
            return target  // 抽出 redirect 目標
        }
    }
    if strings.HasPrefix(rawURL, "/") { return "https://www.google.com" + rawURL }
    return rawURL
}
```

Google Maps 有時把外部連結包成 `/url?q=http://example.com/&opi=...` 形式，這函式把真實 URL 挖出來。

### 7.2 兩層 email 抽取（`emailjob.go:64-98, 104-139`）

```go
func (j *EmailExtractJob) Process(ctx context.Context, resp *scrapemate.Response) (any, []scrapemate.IJob, error) {
    defer func() {
        if j.ExitMonitor != nil && !j.WriterManagedCompletion {
            j.ExitMonitor.IncrPlacesCompleted(1)   // ★ 完工記錄在 defer 裡，連 error 也算
        }
    }()

    if resp.Error != nil { return j.Entry, nil, nil }  // 連 fetch 都失敗 → 仍寫出 entry 但 Emails 為空

    doc, _ := resp.Document.(*goquery.Document)
    emails := docEmailExtractor(doc)              // 先試 DOM
    if len(emails) == 0 {
        emails = regexEmailExtractor(resp.Body)   // 沒中再試 regex
    }
    j.Entry.Emails = emails
    return j.Entry, nil, nil
}
```

**`docEmailExtractor`** 找 `<a href="mailto:...">`：

```go
doc.Find("a[href^='mailto:']").Each(func(_ int, s *goquery.Selection) {
    mailto, _ := s.Attr("href")
    value := strings.TrimPrefix(mailto, "mailto:")
    if email, err := getValidEmail(value); err == nil {
        if !seen[email] { emails = append(emails, email); seen[email] = true }
    }
})
```

**`regexEmailExtractor`** 用 `mcnijman/go-emailaddress` 全文掃描：

```go
addresses := emailaddress.Find(body, false)  // false = 不嚴格 DNS 驗證
```

### 7.3 `ProcessOnFetchError() returns true` —— 全鏈路完工保證

```go
func (j *EmailExtractJob) ProcessOnFetchError() bool { return true }
```

這是 scrapemate 的特殊 hook：**即使 HTTP 抓取失敗（404/超時/連線拒絕），仍呼叫 Process**。為什麼？因為 `defer` 裡的 `IncrPlacesCompleted(1)` 必須被執行，否則 exiter 永遠等不到完工訊號，整個 CLI 程式會卡死。同樣的 `ProcessOnFetchError()` 也存在於 GmapJob 與 PlaceJob，這是貫穿整個 job 圖的「**完工帳本完整性保證**」。

---

## 八、Deduper —— 去重的 double-checked locking

`deduper/hashmap.go:16-35`：

```go
func (d *hashmap) AddIfNotExists(_ context.Context, key string) bool {
    d.mux.RLock()
    if _, ok := d.seen[d.hash(key)]; ok {  // [1] 用讀鎖快速檢查
        d.mux.RUnlock()
        return false
    }
    d.mux.RUnlock()

    d.mux.Lock()                            // [2] 升級為寫鎖
    defer d.mux.Unlock()

    if _, ok := d.seen[d.hash(key)]; ok {   // [3] 再次檢查（其他 goroutine 可能搶先寫了）
        return false
    }
    d.seen[d.hash(key)] = struct{}{}
    return true
}
```

**為什麼用 double-checked locking？**

- 一般情況是「已經看過」（很多 PlaceJob 在不同搜尋頁出現）—— 走 [1] 讀鎖，並發很高。
- 真要插入時才升級成寫鎖，[3] 是為了避免 [1] 到 [2] 之間別的 goroutine 已經插入了同樣的 key。

**FNV-64 hash**：`d.hash(key)` 把 URL 字串雜湊成 `uint64`，比直接用 string key 更省記憶體（`map[uint64]` 比 `map[string]` 小很多）—— 碰撞機率約 1/18 quintillion，實務上忽略不計。

---

## 九、Exiter —— 知道什麼時候該停

`exiter/exiter.go` 維護四個計數器與兩個同步原語：

```go
type exiter struct {
    seedCount       int   // 共要跑幾個 GmapJob/SearchJob
    seedCompleted   int   // 已跑完幾個
    placesFound     int   // 已被排隊的 PlaceJob 數量
    placesCompleted int   // 已跑完的 PlaceJob 數量

    mu         *sync.Mutex
    cancelFunc context.CancelFunc
    doneCh     chan struct{}     // buffer 1 的 signal channel
}
```

完工條件統一：

```go
done := e.seedCompleted >= e.seedCount && e.placesCompleted >= e.placesFound
if done {
    select {
    case e.doneCh <- struct{}{}:
    default:                      // 已經塞過了就吞掉，不重複觸發
    }
}
```

**buffer-1 channel + non-blocking send** 是個常見模式:避免多個 goroutine 同時完成都試圖送 signal 時 block。

`Run(ctx)` 阻塞等 doneCh，收到後呼叫 `cancelFunc()` 通知整個系統收工：

```go
func (e *exiter) Run(ctx context.Context) {
    select {
    case <-ctx.Done(): return         // 上層先 cancel（Ctrl-C）
    case <-e.doneCh:                  // 自然完工
        if e.cancelFunc != nil { e.cancelFunc() }
    }
}
```

**為什麼需要 `placesFound` 與 `placesCompleted` 兩個計數？**
因為 `placesFound` 是動態增長的 —— GmapJob 跑完才知道有多少 PlaceJob 要排。如果只看 `placesCompleted == placesFound`，在還沒有 GmapJob 跑完前（兩者都是 0）就會誤觸發。所以一定要搭配 `seedCompleted >= seedCount` 的前置條件。

---

## 十、端到端時序圖（精簡版）

以 `./google-maps-scraper -input queries.txt -results out.csv -email -c 4` 為例，假設 `queries.txt` 有 1 行 `coffee in Berlin`：

```
時間 →

main()
 └─ ParseConfig()           [推斷 RunMode=RunModeFile]
 └─ runnerFactory()         [回傳 fileRunner]
 └─ filerunner.New()
     ├─ setInput()           [打開 queries.txt]
     ├─ setWriters()         [建 csvWriter 寫 out.csv]
     └─ setApp()             [Concurrency=4, WithJS(DisableImages())]
 └─ runner.Run(ctx)
     ├─ dedup := deduper.New()
     ├─ exitMonitor := exiter.New()
     ├─ CreateSeedJobs() → [1 個 GmapJob]
     ├─ exitMonitor.SetSeedCount(1)
     ├─ go exitMonitor.Run(ctx)        ← 阻塞等 doneCh
     └─ app.Start(ctx, seedJobs...)
         ├─ scrapemate worker 1 拿 GmapJob
         │   ├─ BrowserActions:
         │   │   ├─ Goto + clickRejectCookies + WaitForURL
         │   │   ├─ WaitForSelector('div[role="feed"]')
         │   │   └─ scroll(maxDepth=10) → 1.5x 遞增等待
         │   └─ Process:
         │       ├─ goquery 抽 20 個 anchor
         │       ├─ dedup.AddIfNotExists × 20 (假設全新)
         │       ├─ exitMonitor.IncrPlacesFound(20)
         │       ├─ exitMonitor.IncrSeedCompleted(1)
         │       └─ return [20 PlaceJob]
         │
         ├─ workers 1-4 並行拿 PlaceJob
         │   每個：
         │   ├─ BrowserActions:
         │   │   ├─ Goto place URL
         │   │   ├─ extractJSON: 輪詢 APP_INITIALIZATION_STATE，最多 reload 1 次
         │   │   └─ resp.Meta["json"] = raw
         │   └─ Process:
         │       ├─ EntryFromJSON(raw) → entry
         │       ├─ entry.ID = ParentID
         │       ├─ if entry.WebSite valid:
         │       │     ├─ j.UsageInResultststs = false   ← 不寫我，子 job 寫
         │       │     └─ return [EmailExtractJob{Entry: &entry}]
         │       └─ else:
         │             ├─ exitMonitor.IncrPlacesCompleted(1)
         │             └─ return &entry  → CSV 寫一筆
         │
         ├─ workers 1-4 並行跑 EmailExtractJob (純 HTTP, 無瀏覽器)
         │   每個：
         │   ├─ Goto entry.WebSite
         │   └─ Process:
         │       ├─ defer { IncrPlacesCompleted(1) }
         │       ├─ docEmailExtractor → 失敗 → regexEmailExtractor
         │       ├─ entry.Emails = emails
         │       └─ return entry  → CSV 寫一筆
         │
         └─ 當第 20 個 PlaceJob/EmailJob 完工：
             ├─ exitMonitor 看到 seedCompleted(1) >= seedCount(1) && placesCompleted(20) >= placesFound(20)
             ├─ doneCh <- struct{}{}
             ├─ exitMonitor.Run() goroutine 拿到 signal，呼叫 cancel()
             ├─ app.Start ctx.Done() → 自然返回
             └─ Run() return nil

 └─ runnerInstance.Close(ctx)           [關 outfile, 關 app]
 └─ runner.Telemetry().Close()
 └─ os.Exit(0)
```

---

## 十一、核心業務邏輯總結

CLI 模式的整套設計可以歸納成五個原則：

1. **介面實作切換 + 隱式模式推斷**：同一個 `Runner` 介面、同一個 scrapemate 引擎，靠 flag 組合在六種運行情境間切換，**避免「mode flag」的繁瑣**。

2. **Job 圖的責任轉移**：當 PlaceJob 要 spawn EmailJob 接手寫出時，用 `UsageInResultststs = false` 主動關閉自己的寫入旗標。**寫出責任永遠在「鏈尾」的 job**，避免重複輸出。

3. **完工帳本的完整性保證**：`ProcessOnFetchError() returns true`、`defer IncrPlacesCompleted(1)`、`exitMonitor` 的雙計數器，三層機制確保 **無論 job 怎麼失敗都會被記到帳上**，否則整個程式會卡死等不到的 signal。

4. **共享狀態的並發安全**：`dedup` 與 `exitMonitor` 是貫穿整個 Run 的單例，前者用 double-checked locking + FNV-64 hash 抑制重複 PlaceJob，後者用 buffer-1 channel + non-blocking send 觸發退出。

5. **重試與退避的層層防禦**：
   - 滾動：JS setTimeout 等待 + Go WaitForTimeout 1.5x 退避
   - 取 JSON：getRaw 內層 200ms 輪詢 + extractJSON 外層 reload 重試
   - 抓 email：先 DOM 再 regex
   - scrapemate 本身的 `MaxRetries: 3`（GmapJob、PlaceJob）

這就是「**CLI 一行命令 → 一個 CSV**」表面下的完整實作。
