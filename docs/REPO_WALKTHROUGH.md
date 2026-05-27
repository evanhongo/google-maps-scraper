# Google Maps Scraper — A Linear Code Walkthrough

*2026-05-21T16:31:35Z by Showboat dev*
<!-- showboat-id: 2c4b08b3-83a6-4f77-a31f-151eeb7161d4 -->

This document is a top-to-bottom tour of the `google-maps-scraper` Go codebase. The aim is to leave a reader who has read it (and only it) able to navigate, reason about, and modify any part of the project without further exploration.

The project is, at its core, a single Go binary (`./main.go`) that scrapes business listings from Google Maps. What makes it interesting is that the *same* binary can be operated in **six** very different ways: as a CLI that writes CSV, as a PostgreSQL-backed batch worker, as a local web UI, as an AWS Lambda function, as the dispatcher that *invokes* those Lambdas, and as an installer that downloads Playwright browsers. On top of all that, there is a separate binary (`cmd/gmapssaas`) for the multi-tenant "SaaS Edition" that adds an admin panel, REST API, River-queue job dispatch, and infrastructure-provisioning helpers.

We are going to walk through the code roughly in the order that a request flows through it:

1. The entry point and configuration system.
2. The `Runner` interface and the six concrete runners.
3. The core scraping engine in `gmaps/` — what a job is, how the browser is driven, how a place page is parsed into an `Entry`.
4. Cross-cutting helpers (seed-job construction, the `deduper`, the `exiter`, grid scraping).
5. Output writers — CSV/JSON, PostgreSQL, LeadsDB, the central writer used by the SaaS edition.
6. The local web/SQLite runner and its REST API.
7. The SaaS Edition: River queue, scraper manager, admin panel, REST API, infrastructure provisioning.
8. Build, telemetry, and packaging.

Code snippets are pulled directly from the repository at the commit you are reading this on; running `showboat verify docs/REPO_WALKTHROUGH.md` will re-execute every command and warn if anything has drifted.

## 1. Project layout

Before diving into code, here's the top level. Treat this as the map you'll mentally annotate as we go.

```bash
find . -maxdepth 1 -type d -not -path '.' -not -path './.git' -not -path './.github' | sort
```

```output
./admin
./api
./cli
./cmd
./cryptoext
./deduper
./docs
./env
./examples
./exiter
./gmaps
./grid
./httpext
./img
./infra
./internal
./leadsdb
./local
./log
./migrations
./postgres
./ratelimit
./rqueue
./runner
./s3uploader
./saas
./scraper
./scripts
./skills
./sponsors
./testdata
./tlmt
./web
```

At a glance:

| Package | What lives there |
|---|---|
| `runner/` | The `Runner` interface, configuration parsing, and the six concrete run modes (`filerunner`, `webrunner`, `databaserunner`, `lambdaaws`, `installplaywright`). Also `jobs.go`, which builds seed jobs from query files. |
| `gmaps/` | The actual scraping logic. Job structs (`GmapJob`, `PlaceJob`, `EmailExtractJob`, `SearchJob`), the `Entry` result struct, JSON parsing of Google's `APP_INITIALIZATION_STATE`, and the extra-reviews path. |
| `grid/` | Bounding-box → cells utility used to bypass Google's ~120-result-per-search cap by tiling the area into many overlapping searches. |
| `deduper/`, `exiter/` | Tiny utilities. The deduper is a thread-safe FNV-hashed seen-set; the exiter is the bookkeeper that decides when "all jobs are done" and cancels the root context. |
| `web/` | The local web UI runner — a small Go `net/http` server, SQLite-backed job queue, embedded HTML templates, and a JSON REST API. |
| `postgres/` | A `scrapemate.JobProvider` and `ResultWriter` that store/read jobs and results in PostgreSQL (used by `databaserunner` for distributed workers). |
| `leadsdb/` | An optional `ResultWriter` that batches entries into the third-party LeadsDB service. |
| `migrations/` | Embedded SQL migrations for the SaaS Postgres schema. |
| `tlmt/` | Anonymous PostHog telemetry. Disabled with `DISABLE_TELEMETRY=1`. |
| `s3uploader/` | Thin wrapper around the AWS SDK used by the Lambda runner. |
| `cmd/gmapssaas/` | The **SaaS Edition** binary — a separate `main` package with subcommands `serve`, `worker`, `provision`, `admin`, `update`. |
| `admin/`, `api/`, `rqueue/`, `scraper/` | The pieces of the SaaS Edition: admin UI, REST API, [River](https://riverqueue.com/) queue integration, and the scraper-manager lifecycle. |
| `infra/`, `cli/` | Cloud provisioning (DigitalOcean, Hetzner, PlanetScale) for the SaaS Edition and shared CLI helpers. |
| `internal/jsonbsanitize/` | Strips NUL bytes from entries so PostgreSQL JSONB inserts don't fail. |
| `cryptoext/`, `ratelimit/`, `httpext/`, `log/`, `env/` | Cross-cutting helpers — encryption, rate limiting, HTTP middleware, structured logging, env-var loading. |
| `skills/` | An [Anthropic skill](https://docs.anthropic.com/) that teaches an AI agent how to run this scraper inside Claude Code. |

## 2. The entry point — `main.go`

Everything starts at `main()`. It does three things: install a SIGINT/SIGTERM handler so Ctrl-C is graceful, call `runner.ParseConfig()` to decide which mode we're in, and hand off to `runnerFactory()`. Then it just calls `Run` on whatever runner was selected and waits.

```bash
sed -n '20,65p' main.go
```

```output
func main() {
	ctx, cancel := context.WithCancel(context.Background())

	runner.Banner()

	sigChan := make(chan os.Signal, 1)
	signal.Notify(sigChan, os.Interrupt, syscall.SIGTERM)

	go func() {
		<-sigChan

		log.Println("Received signal, shutting down...")

		cancel()
	}()

	cfg := runner.ParseConfig()

	runnerInstance, err := runnerFactory(cfg)
	if err != nil {
		cancel()
		os.Stderr.WriteString(err.Error() + "\n")

		runner.Telemetry().Close()

		os.Exit(1)
	}

	if err := runnerInstance.Run(ctx); err != nil && !errors.Is(err, context.Canceled) {
		os.Stderr.WriteString(err.Error() + "\n")

		_ = runnerInstance.Close(ctx)
		runner.Telemetry().Close()

		cancel()

		os.Exit(1)
	}

	_ = runnerInstance.Close(ctx)
	runner.Telemetry().Close()

	cancel()

	os.Exit(0)
}
```

`runnerFactory` is a tidy switch over `cfg.RunMode`. Each mode constructs a different concrete implementation of `runner.Runner`. This is the central dispatch table for the whole binary; once you understand which factory is chosen, you know which of the six worlds you're in.

```bash
sed -n '67,84p' main.go
```

```output
func runnerFactory(cfg *runner.Config) (runner.Runner, error) {
	switch cfg.RunMode {
	case runner.RunModeFile:
		return filerunner.New(cfg)
	case runner.RunModeDatabase, runner.RunModeDatabaseProduce:
		return databaserunner.New(cfg)
	case runner.RunModeInstallPlaywright:
		return installplaywright.New(cfg)
	case runner.RunModeWeb:
		return webrunner.New(cfg)
	case runner.RunModeAwsLambda:
		return lambdaaws.New(cfg)
	case runner.RunModeAwsLambdaInvoker:
		return lambdaaws.NewInvoker(cfg)
	default:
		return nil, fmt.Errorf("%w: %d", runner.ErrInvalidRunMode, cfg.RunMode)
	}
}
```

## 3. Configuration — `runner/runner.go`

`ParseConfig()` is the longest function in the package. It binds every CLI flag, applies environment-variable fallbacks, runs basic validation, and finally decides `cfg.RunMode` based on which flags were set. The interesting part is *how* it picks a mode — there are no `--mode` flags; the mode is implied by the combination of inputs.

The `Config` struct lives at the top of the file. It's a plain bag of CLI flag values, plus a couple of computed fields (`Proxies` parsed from comma-separated input, `S3Uploader` built when AWS creds are present).

```bash
sed -n '25,33p' runner/runner.go
```

```output
const (
	RunModeFile = iota + 1
	RunModeDatabase
	RunModeDatabaseProduce
	RunModeInstallPlaywright
	RunModeWeb
	RunModeAwsLambda
	RunModeAwsLambdaInvoker
)
```

The dispatch logic at the bottom of `ParseConfig` is worth reading carefully. Note the precedence: AWS Lambda flags trump everything; otherwise web mode is the default when neither a DSN nor an input file is given (this makes a no-args `./google-maps-scraper` launch the web UI, which is what most users want); otherwise file mode unless a Postgres DSN is set; and a `--produce` flag with a DSN means "stuff seed jobs into Postgres and exit".

```bash
sed -n '214,232p' runner/runner.go
```

```output
	switch {
	case cfg.AwsLambdaInvoker:
		cfg.RunMode = RunModeAwsLambdaInvoker
	case cfg.AwsLamdbaRunner:
		cfg.RunMode = RunModeAwsLambda
	case cfg.WebRunner || (cfg.Dsn == "" && cfg.InputFile == ""):
		cfg.RunMode = RunModeWeb
	case cfg.Dsn == "":
		cfg.RunMode = RunModeFile
	case cfg.ProduceOnly:
		cfg.RunMode = RunModeDatabaseProduce
	case cfg.Dsn != "":
		cfg.RunMode = RunModeDatabase
	default:
		panic("Invalid configuration")
	}

	return &cfg
}
```

The `Runner` interface itself is small: a `Run(ctx)` that does the work and a `Close(ctx)` that releases resources. Every concrete runner satisfies this contract.

```bash
sed -n '39,47p' runner/runner.go
```

```output
type Runner interface {
	Run(context.Context) error
	Close(context.Context) error
}

type S3Uploader interface {
	Upload(ctx context.Context, bucketName, key string, body io.Reader) error
}

```

## 4. The scraping engine — `gmaps/`

This is the heart of the project. Everything else is plumbing that schedules jobs from this package, collects results, and writes them somewhere. The engine sits on top of [scrapemate](https://github.com/gosom/scrapemate), the author's own scraping framework. ScrapeMate gives you three primitives — `Job`, `JobProvider` (a queue), and `ResultWriter` (a sink) — and a pool of headless-browser workers that pull jobs and call their `Process` method with the HTTP response. We define jobs; scrapemate runs them.

There are **four** job types, with a parent-child relationship:

<pre>
GmapJob (search results page)
   │   one per query
   │
   ├──> PlaceJob (place detail page)
   │       │   one per place URL extracted from the list
   │       │
   │       └──> EmailExtractJob  (visits entry.WebSite to find emails)
   │              one per place that has a website, only if -email is set
   │
   └──> (alternative path)
        SearchJob   used in "fast mode" — HTTP-only, no browser
</pre>

`GmapJob` and `PlaceJob` need a browser (Playwright via scrapemate's JS option) because Google Maps is a heavy SPA. `EmailExtractJob` is plain HTTP — it visits the business's website and scrapes `mailto:` links plus regex hits. `SearchJob` is the fast-mode fallback: it hits the internal `maps.google.com/search` JSON endpoint directly with a `tbm=map` and a hand-crafted `pb` parameter, and parses the protobuf-shaped response with no browser at all.

### 4.1 The `Entry` struct — what we're producing

Everything we write out is an `Entry`. Reading this struct first gives you context for everything else: this is the shape of one row of CSV, or one record of JSON.

```bash
sed -n '60,98p' gmaps/entry.go
```

```output
type Entry struct {
	ID         string              `json:"input_id"`
	Link       string              `json:"link"`
	Cid        string              `json:"cid"`
	Title      string              `json:"title"`
	Categories []string            `json:"categories"`
	Category   string              `json:"category"`
	Address    string              `json:"address"`
	OpenHours  map[string][]string `json:"open_hours"`
	// PopularTImes is a map with keys the days of the week
	// and value is a map with key the hour and value the traffic in that time
	PopularTimes        map[string]map[int]int `json:"popular_times"`
	WebSite             string                 `json:"web_site"`
	Phone               string                 `json:"phone"`
	PlusCode            string                 `json:"plus_code"`
	ReviewCount         int                    `json:"review_count"`
	ReviewRating        float64                `json:"review_rating"`
	ReviewsPerRating    map[int]int            `json:"reviews_per_rating"`
	Latitude            float64                `json:"latitude"`
	Longtitude          float64                `json:"longtitude"`
	Status              string                 `json:"status"`
	Description         string                 `json:"description"`
	ReviewsLink         string                 `json:"reviews_link"`
	Thumbnail           string                 `json:"thumbnail"`
	Timezone            string                 `json:"timezone"`
	PriceRange          string                 `json:"price_range"`
	DataID              string                 `json:"data_id"`
	PlaceID             string                 `json:"place_id"`
	Images              []Image                `json:"images"`
	Reservations        []LinkSource           `json:"reservations"`
	OrderOnline         []LinkSource           `json:"order_online"`
	Menu                LinkSource             `json:"menu"`
	Owner               Owner                  `json:"owner"`
	CompleteAddress     Address                `json:"complete_address"`
	About               []About                `json:"about"`
	UserReviews         []Review               `json:"user_reviews"`
	UserReviewsExtended []Review               `json:"user_reviews_extended"`
	Emails              []string               `json:"emails"`
}
```

A few fields are worth calling out: `ID` is the *parent* job's ID, not a unique-per-row identifier — every place found through the same input query line carries the same `ID`, which is what lets the `#!#` suffix in the input file (see below) flow through to the output. `UserReviews` is whatever was in the JSON blob from the place page (always a small sample). `UserReviewsExtended` is the full list, populated only when `--extra-reviews` was passed, and it can come from one of two sources (more on that in 4.6). `Emails` is filled in by `EmailExtractJob` only if `--email` is set.

### 4.2 The job hierarchy — `GmapJob` → `PlaceJob` → `EmailExtractJob`

A `GmapJob` is a search-results page. Its URL is `https://www.google.com/maps/search/<query>[/@lat,lon,Nz]`. When it runs in a browser, it scrolls the side panel (up to `MaxDepth` times) so Google lazy-loads more results, then we extract the place URLs from the rendered DOM.

```bash
sed -n '34,80p' gmaps/job.go
```

```output
func NewGmapJob(
	id, langCode, query string,
	maxDepth int,
	extractEmail bool,
	geoCoordinates string,
	zoom int,
	opts ...GmapJobOptions,
) *GmapJob {
	query = url.QueryEscape(query)

	const (
		maxRetries = 3
		prio       = scrapemate.PriorityLow
	)

	if id == "" {
		id = uuid.New().String()
	}

	mapURL := ""
	if geoCoordinates != "" && zoom > 0 {
		mapURL = fmt.Sprintf("https://www.google.com/maps/search/%s/@%s,%dz", query, strings.ReplaceAll(geoCoordinates, " ", ""), zoom)
	} else {
		// Warning: geo and zoom MUST be both set or not
		mapURL = fmt.Sprintf("https://www.google.com/maps/search/%s", query)
	}

	job := GmapJob{
		Job: scrapemate.Job{
			ID:         id,
			Method:     http.MethodGet,
			URL:        mapURL,
			URLParams:  map[string]string{"hl": langCode},
			MaxRetries: maxRetries,
			Priority:   prio,
		},
		MaxDepth:     maxDepth,
		LangCode:     langCode,
		ExtractEmail: extractEmail,
	}

	for _, opt := range opts {
		opt(&job)
	}

	return &job
}
```

A scrapemate job has two halves: `BrowserActions(ctx, page)` is called *first* and returns a `Response` (typically by running JavaScript in the browser and harvesting the resulting HTML); then `Process(ctx, resp)` is called with that response and returns either result data, more jobs to enqueue, or both. The split is deliberate — the browser-bound half runs in scrapemate's bounded browser pool, the post-processing half runs in a CPU-bound pool, so we don't tie up a Chromium tab while parsing.

For `GmapJob`, `BrowserActions` does the scrolling, and `Process` does two things: if the resulting URL is a `/maps/place/` URL (Google redirected our search to a single result), we go straight to a `PlaceJob` for it; otherwise we read the rendered DOM and emit one `PlaceJob` per link in the side-feed.

```bash
sed -n '114,184p' gmaps/job.go
```

```output
func (j *GmapJob) Process(ctx context.Context, resp *scrapemate.Response) (any, []scrapemate.IJob, error) {
	defer func() {
		resp.Document = nil
		resp.Body = nil
	}()

	if resp.Error != nil {
		if j.ExitMonitor != nil {
			j.ExitMonitor.IncrSeedCompleted(1)
		}

		return nil, nil, resp.Error
	}

	log := scrapemate.GetLoggerFromContext(ctx)

	doc, ok := resp.Document.(*goquery.Document)
	if !ok {
		if j.ExitMonitor != nil {
			j.ExitMonitor.IncrSeedCompleted(1)
		}

		return nil, nil, fmt.Errorf("could not convert to goquery document")
	}

	var next []scrapemate.IJob

	if strings.Contains(resp.URL, "/maps/place/") {
		jopts := []PlaceJobOptions{}
		if j.ExitMonitor != nil {
			jopts = append(jopts, WithPlaceJobExitMonitor(j.ExitMonitor))
		}

		if j.WriterManagedCompletion {
			jopts = append(jopts, WithPlaceJobWriterManagedCompletion())
		}

		placeJob := NewPlaceJob(j.ID, j.LangCode, resp.URL, j.ExtractEmail, j.ExtractExtraReviews, jopts...)

		next = append(next, placeJob)
	} else {
		doc.Find(`div[role=feed] div[jsaction]>a`).Each(func(_ int, s *goquery.Selection) {
			if href := s.AttrOr("href", ""); href != "" {
				jopts := []PlaceJobOptions{}
				if j.ExitMonitor != nil {
					jopts = append(jopts, WithPlaceJobExitMonitor(j.ExitMonitor))
				}

				if j.WriterManagedCompletion {
					jopts = append(jopts, WithPlaceJobWriterManagedCompletion())
				}

				nextJob := NewPlaceJob(j.ID, j.LangCode, href, j.ExtractEmail, j.ExtractExtraReviews, jopts...)

				if j.Deduper == nil || j.Deduper.AddIfNotExists(ctx, href) {
					next = append(next, nextJob)
				}
			}
		})
	}

	if j.ExitMonitor != nil {
		j.ExitMonitor.IncrPlacesFound(len(next))
		j.ExitMonitor.IncrSeedCompleted(1)
	}

	log.Info(fmt.Sprintf("%d places found", len(next)))

	return nil, next, nil
}

```

Note `GmapJob.UseInResults()` returns `false` — the search-results page never produces an `Entry` directly, only child jobs. That's important to understand because it affects ScrapeMate's accounting: a `GmapJob` finishing produces zero rows, only `PlaceJob` finishing produces rows. The `ExitMonitor` is told `IncrPlacesFound(N)` so it can later wait for `N` `IncrPlacesCompleted` callbacks before the run can exit.

The scrolling itself is JavaScript-driven. We grab the side-feed element, set `scrollTop = scrollHeight`, wait, and check whether the height grew. If not, we stop. The wait time grows non-linearly to give slower networks more breathing room.

```bash
sed -n '300,373p' gmaps/job.go
```

```output
func scroll(ctx context.Context,
	page scrapemate.BrowserPage,
	maxDepth int,
	scrollSelector string,
) (int, error) {
	expr := `async () => {
		const el = document.querySelector("` + scrollSelector + `");
		el.scrollTop = el.scrollHeight;

		return new Promise((resolve, reject) => {
  			setTimeout(() => {
    		resolve(el.scrollHeight);
  			}, %d);
		});
	}`

	var currentScrollHeight int
	// Scroll to the bottom of the page.
	waitTime := 100.
	cnt := 0

	const (
		timeout  = 500
		maxWait2 = 2000
	)

	for i := 0; i < maxDepth; i++ {
		cnt++
		waitTime2 := timeout * cnt

		if waitTime2 > timeout {
			waitTime2 = maxWait2
		}

		// Scroll to the bottom of the page.
		scrollHeight, err := page.Eval(fmt.Sprintf(expr, waitTime2))
		if err != nil {
			return cnt, err
		}

		// Handle both int and float64 because browser-evaluated numbers may arrive as either type.
		var height int
		switch v := scrollHeight.(type) {
		case int:
			height = v
		case float64:
			height = int(v)
		default:
			return cnt, fmt.Errorf("scrollHeight is not a number, got %T", scrollHeight)
		}

		if height == currentScrollHeight {
			break
		}

		currentScrollHeight = height

		select {
		case <-ctx.Done():
			return currentScrollHeight, nil
		default:
		}

		waitTime *= 1.5

		if waitTime > maxWait2 {
			waitTime = maxWait2
		}

		page.WaitForTimeout(time.Duration(waitTime) * time.Millisecond)
	}

	return cnt, nil
}
```

### 4.3 `PlaceJob` — extracting one business

`PlaceJob` visits a single place URL and pulls a JSON blob out of `window.APP_INITIALIZATION_STATE`. Google Maps hydrates the page with a giant data array; we walk through it from JavaScript looking for an entry that starts with the magic `)]}'` prefix — that's the convention Google uses to mark JSON-that-must-not-be-eval'd.

```bash
sed -n '297,319p' gmaps/place.go
```

```output
const js = `
(function() {
	if (!window.APP_INITIALIZATION_STATE || !window.APP_INITIALIZATION_STATE[3]) {
		return null;
	}
	const appState = window.APP_INITIALIZATION_STATE[3];
	
	// Search all properties of appState for arrays containing JSON strings
	for (const key of Object.keys(appState)) {
		const arr = appState[key];
		if (Array.isArray(arr)) {
			// Check indices 6 and 5 (where place data typically is)
			for (const idx of [6, 5]) {
				const item = arr[idx];
				if (typeof item === 'string' && item.startsWith(")]}'")) {
					return item;
				}
			}
		}
	}
	return null;
})()
`
```

That blob is the source of truth for everything in the `Entry`. `EntryFromJSON` parses it (in `gmaps/entry.go`) and returns a populated struct. `PlaceJob.Process` then has a single decision to make: if `-email` is set *and* the entry has a sensible website, fan out one `EmailExtractJob` (carrying the entry by pointer) and return no result yet; otherwise return the entry directly to be written.

```bash
sed -n '72,144p' gmaps/place.go
```

```output
func (j *PlaceJob) Process(_ context.Context, resp *scrapemate.Response) (any, []scrapemate.IJob, error) {
	defer func() {
		resp.Document = nil
		resp.Body = nil
		resp.Meta = nil
	}()

	if resp.Error != nil {
		if j.ExitMonitor != nil {
			j.ExitMonitor.IncrPlacesCompleted(1)
		}

		return nil, nil, resp.Error
	}

	raw, ok := resp.Meta["json"].([]byte)
	if !ok {
		if j.ExitMonitor != nil {
			j.ExitMonitor.IncrPlacesCompleted(1)
		}

		return nil, nil, fmt.Errorf("could not convert to []byte")
	}

	entry, err := EntryFromJSON(raw)
	if err != nil {
		if j.ExitMonitor != nil {
			j.ExitMonitor.IncrPlacesCompleted(1)
		}

		return nil, nil, err
	}

	entry.ID = j.ParentID

	if entry.Link == "" {
		entry.Link = j.GetURL()
	}

	// Handle RPC-based reviews
	allReviewsRaw, ok := resp.Meta["reviews_raw"].(FetchReviewsResponse)
	if ok && len(allReviewsRaw.pages) > 0 {
		entry.AddExtraReviews(allReviewsRaw.pages)
	}

	// Handle DOM-based reviews (fallback)
	domReviews, ok := resp.Meta["dom_reviews"].([]DOMReview)
	if ok && len(domReviews) > 0 {
		convertedReviews := ConvertDOMReviewsToReviews(domReviews)
		entry.UserReviewsExtended = append(entry.UserReviewsExtended, convertedReviews...)
	}

	if j.ExtractEmail && entry.IsWebsiteValidForEmail() {
		opts := []EmailExtractJobOptions{}
		if j.ExitMonitor != nil {
			opts = append(opts, WithEmailJobExitMonitor(j.ExitMonitor))
		}

		if j.WriterManagedCompletion {
			opts = append(opts, WithEmailJobWriterManagedCompletion())
		}

		emailJob := NewEmailJob(j.ID, &entry, opts...)

		j.UsageInResultststs = false

		return nil, []scrapemate.IJob{emailJob}, nil
	} else if j.ExitMonitor != nil && !j.WriterManagedCompletion {
		j.ExitMonitor.IncrPlacesCompleted(1)
	}

	return &entry, nil, err
}
```

The clever bit is `j.UsageInResultststs = false` — `PlaceJob.UseInResults()` consults that field. By setting it false when we spin off an email job, we tell the scrapemate writer "don't write me, the child job will own this entry." The `EmailExtractJob` then fills in `entry.Emails` and writes the same struct itself.

### 4.4 `EmailExtractJob` — fishing for emails on the business website

Email extraction is best-effort and intentionally lenient. The job visits `entry.WebSite`, then tries two extractors in order: DOM-based (look for `<a href="mailto:...">` links) and, if that finds nothing, a regex sweep of the raw response body via `github.com/mcnijman/go-emailaddress`.

```bash
sed -n '64,98p' gmaps/emailjob.go
```

```output
func (j *EmailExtractJob) Process(ctx context.Context, resp *scrapemate.Response) (any, []scrapemate.IJob, error) {
	defer func() {
		resp.Document = nil
		resp.Body = nil
	}()

	defer func() {
		if j.ExitMonitor != nil && !j.WriterManagedCompletion {
			j.ExitMonitor.IncrPlacesCompleted(1)
		}
	}()

	log := scrapemate.GetLoggerFromContext(ctx)

	log.Info("Processing email job", "url", j.URL)

	// if html fetch failed just return
	if resp.Error != nil {
		return j.Entry, nil, nil
	}

	doc, ok := resp.Document.(*goquery.Document)
	if !ok {
		return j.Entry, nil, nil
	}

	emails := docEmailExtractor(doc)
	if len(emails) == 0 {
		emails = regexEmailExtractor(resp.Body)
	}

	j.Entry.Emails = emails

	return j.Entry, nil, nil
}
```

Notice the `ProcessOnFetchError() bool { return true }` defined a few lines below — this is a scrapemate convention. By returning `true`, this job is processed even when the HTTP fetch itself failed, which lets us still emit the place entry (just with `Emails == nil`) rather than dropping it. The deferred `IncrPlacesCompleted(1)` is what eventually unblocks the exiter and lets the run finish.

### 4.5 `SearchJob` — the fast-mode alternative

When `--fast-mode` is set together with `--geo` coordinates, we skip the browser entirely and call Google's internal Maps search endpoint. The URL is `https://maps.google.com/search?tbm=map&pb=<long-protobuf-shaped-thing>`, with the lat/lon/zoom/viewport baked into the `pb` parameter.

```bash
sed -n '146,171p' gmaps/searchjob.go
```

```output
func buildGoogleMapsParams(params *MapSearchParams) map[string]string {
	params.ViewportH = 800
	params.ViewportW = 600

	ans := map[string]string{
		"tbm":      "map",
		"authuser": "0",
		"hl":       params.Hl,
		"q":        params.Query,
	}

	pb := fmt.Sprintf("!4m12!1m3!1d3826.902183192154!2d%.4f!3d%.4f!2m3!1f0!2f0!3f0!3m2!1i%d!2i%d!4f%.1f!7i20!8i0"+
		"!10b1!12m22!1m3!18b1!30b1!34e1!2m3!5m1!6e2!20e3!4b0!10b1!12b1!13b1!16b1!17m1!3e1!20m3!5e2!6b1!14b1!46m1!1b0"+
		"!96b1!19m4!2m3!1i360!2i120!4i8",
		params.Location.Lon,
		params.Location.Lat,
		params.ViewportW,
		params.ViewportH,
		params.Location.ZoomLvl,
	)

	ans["pb"] = pb

	return ans
}
```

The response body starts with one junk line that has to be stripped (`removeFirstLine`), and the rest is parsed in `gmaps/multiple.go` by `ParseSearchResults`. The output is `[]Entry` — no second-stage `PlaceJob` is needed because the search response contains enough data per result to populate an `Entry` directly. The trade-off: no website-scraping for emails, no extra reviews, slightly less data than the browser path, but vastly faster and lighter on resources. Fast mode also runs through scrapemate's `WithStealth("firefox")` to evade anti-bot heuristics.

### 4.6 Extra reviews — RPC and DOM fallback

`PlaceJob.BrowserActions` (in `gmaps/place.go`) optionally pulls every review for a place when `--extra-reviews` is set. The strategy is in `gmaps/reviews.go`: try the RPC endpoint that Google uses internally first; if that fails or returns nothing, fall back to clicking around in the DOM. Both paths feed into `entry.UserReviewsExtended`.

```bash
sed -n '180,205p' gmaps/place.go
```

```output
	if j.ExtractExtraReviews {
		reviewCount := j.getReviewCount(raw)
		if reviewCount > 0 { // download reviews for any place that has them
			params := fetchReviewsParams{
				page:        page,
				mapURL:      page.URL(),
				reviewCount: reviewCount,
			}

			// Use the new fallback mechanism that tries RPC first, then DOM
			rpcData, domReviews, err := FetchReviewsWithFallback(ctx, params)

			switch {
			case err != nil:
				fmt.Printf("Warning: review extraction failed: %v\n", err)
			case len(rpcData.pages) > 0:
				resp.Meta["reviews_raw"] = rpcData
			case len(domReviews) > 0:
				resp.Meta["dom_reviews"] = domReviews
			}
		}
	}

	return resp
}

```

## 5. Building seed jobs — `runner/jobs.go`

A seed job is the entry point of the job graph for a single input query. `CreateSeedJobs` reads queries one per line from a file (or stdin), produces a `GmapJob` (or `SearchJob` in fast mode) for each, and returns the slice. The shared `deduper` and `exitMonitor` are wired into every job, which is what makes them act as run-wide singletons.

Each input line can have an optional ID via the `#!#` separator — so `coffee shops in NYC#!#nyc-coffee` produces a job whose `ID` is `nyc-coffee` and that ID is propagated to every `Entry` that comes out of it. Anything after the first `#!#` is the ID; lines without one get a random UUID. Look at the `parseQueryLine` helper to see the contract exactly.

```bash
sed -n '245,266p' runner/jobs.go
```

```output
func parseQueryLine(line string) (query, bool, error) {
	line = strings.TrimSpace(line)
	if line == "" {
		return query{}, false, nil
	}

	var q query

	if before, after, ok := strings.Cut(line, "#!#"); ok {
		q.text = strings.TrimSpace(before)
		q.id = strings.TrimSpace(after)
	} else {
		q.text = line
	}

	if q.text == "" {
		return query{}, false, fmt.Errorf("invalid query line %q: empty query text", line)
	}

	return q, true, nil
}

```

There's also a `LoadCustomWriter(pluginDir, pluginName)` at the bottom of the same file — it `plugin.Open`s a `.so` (Linux/Mac) or `.dll` (Windows) and looks up a symbol that the user names with `-writer dir:pluginName`. The symbol must be a `*scrapemate.ResultWriter`. This is how someone can ship their own output sink without forking the project.

## 6. Cross-cutting utilities — `deduper` and `exiter`

Two small packages that show up in every runner.

### 6.1 `deduper` — a thread-safe seen-URL set

`Deduper.AddIfNotExists` returns `true` only the first time it sees a key. The implementation hashes the key with FNV-64 and stores it in a `map[uint64]struct{}` behind a `sync.RWMutex`. This is used everywhere a `GmapJob` is about to enqueue a `PlaceJob` — without it, the same place URL appearing in multiple result pages (or in overlapping grid cells) would be scraped twice.

```bash
sed -n '11,43p' deduper/hashmap.go
```

```output
type hashmap struct {
	mux  *sync.RWMutex
	seen map[uint64]struct{}
}

func (d *hashmap) AddIfNotExists(_ context.Context, key string) bool {
	d.mux.RLock()
	if _, ok := d.seen[d.hash(key)]; ok {
		d.mux.RUnlock()
		return false
	}

	d.mux.RUnlock()

	d.mux.Lock()
	defer d.mux.Unlock()

	if _, ok := d.seen[d.hash(key)]; ok {
		return false
	}

	d.seen[d.hash(key)] = struct{}{}

	return true
}

func (d *hashmap) hash(key string) uint64 {
	h := fnv.New64()
	h.Write([]byte(key))

	return h.Sum64()
}
```

The double-checked locking pattern (RLock first, then Lock + re-check) is a deliberate optimization — the common case is "already seen", so we want the fast read path to take only an RLock.

### 6.2 `exiter` — knowing when the run is done

scrapemate doesn't know when *your* job graph is finished — it just pulls jobs from the provider until told to stop. The exiter tracks four counters and cancels the parent `context.Context` once everything checks out: `seedCount` (how many `GmapJob`s/`SearchJob`s we created), `seedCompleted` (how many have run), `placesFound` (how many `PlaceJob`s were spawned in total), `placesCompleted` (how many of those finished). When `seedCompleted >= seedCount && placesCompleted >= placesFound`, we're done.

```bash
sed -n '49,94p' exiter/exiter.go
```

```output
func (e *exiter) IncrSeedCompleted(val int) {
	e.mu.Lock()
	e.seedCompleted += val
	done := e.seedCompleted >= e.seedCount && e.placesCompleted >= e.placesFound
	e.mu.Unlock()

	if done {
		select {
		case e.doneCh <- struct{}{}:
		default:
		}
	}
}

func (e *exiter) IncrPlacesFound(val int) {
	e.mu.Lock()
	defer e.mu.Unlock()

	e.placesFound += val
}

func (e *exiter) IncrPlacesCompleted(val int) {
	e.mu.Lock()
	e.placesCompleted += val
	done := e.seedCompleted >= e.seedCount && e.placesCompleted >= e.placesFound
	e.mu.Unlock()

	if done {
		select {
		case e.doneCh <- struct{}{}:
		default:
		}
	}
}

func (e *exiter) Run(ctx context.Context) {
	select {
	case <-ctx.Done():
		return
	case <-e.doneCh:
		if e.cancelFunc != nil {
			e.cancelFunc()
		}
	}
}
```

## 7. The simplest runner — `runner/filerunner/filerunner.go`

With the engine pieces in hand, the file runner is now easy to read end-to-end. It reads queries, builds writers (CSV, JSON, custom plugin, or LeadsDB), builds a scrapemate app with the right options, builds the seed jobs (with grid support if `--grid-bbox` is set), wires up the exiter, then calls `app.Start(ctx, seedJobs...)` and waits.

```bash
sed -n '56,137p' runner/filerunner/filerunner.go
```

```output
func (r *fileRunner) Run(ctx context.Context) (err error) {
	var seedJobs []scrapemate.IJob

	t0 := time.Now().UTC()

	defer func() {
		elapsed := time.Now().UTC().Sub(t0)
		params := map[string]any{
			"job_count": len(seedJobs),
			"duration":  elapsed.String(),
		}

		if err != nil {
			params["error"] = err.Error()
		}

		evt := tlmt.NewEvent("file_runner", params)

		_ = runner.Telemetry().Send(ctx, evt)
	}()

	dedup := deduper.New()
	exitMonitor := exiter.New()

	if r.cfg.GridBBox != "" {
		if r.cfg.FastMode {
			return fmt.Errorf("-fast-mode cannot be used together with -grid-bbox")
		}

		bbox, bboxErr := grid.ParseBoundingBox(r.cfg.GridBBox)
		if bboxErr != nil {
			return fmt.Errorf("invalid -grid-bbox: %w", bboxErr)
		}

		cellCount := grid.EstimateCellCount(bbox, r.cfg.GridCellKm)
		fmt.Fprintf(os.Stderr, "grid scraping: ~%d cells (%.2f km each)\n", cellCount, r.cfg.GridCellKm)

		seedJobs, err = runner.CreateGridSeedJobs(
			r.cfg.LangCode,
			r.input,
			r.cfg.MaxDepth,
			r.cfg.Email,
			bbox,
			r.cfg.GridCellKm,
			r.cfg.Zoom,
			dedup,
			exitMonitor,
			r.cfg.ExtraReviews,
		)
	} else {
		seedJobs, err = runner.CreateSeedJobs(
			r.cfg.FastMode,
			r.cfg.LangCode,
			r.input,
			r.cfg.MaxDepth,
			r.cfg.Email,
			r.cfg.GeoCoordinates,
			r.cfg.Zoom,
			r.cfg.Radius,
			dedup,
			exitMonitor,
			r.cfg.ExtraReviews,
		)
	}

	if err != nil {
		return err
	}

	exitMonitor.SetSeedCount(len(seedJobs))

	ctx, cancel := context.WithCancel(ctx)
	defer cancel()

	exitMonitor.SetCancelFunc(cancel)

	go exitMonitor.Run(ctx)

	err = r.app.Start(ctx, seedJobs...)

	return err
}
```

`setApp` configures scrapemate. Two switches are worth knowing about: `WithJS` puts a browser in front of every fetch (slower but mandatory for the non-fast paths), `WithStealth("firefox")` is the fast-mode alternative that uses a stealth HTTP client with TLS-fingerprinting tricks. `WithPageReuseLimit` is called twice in the file runner — that's not a bug, scrapemate's API accepts repeated options and the second call (200) wins; this is the maximum number of `Page` instances kept warm in the browser pool.

## 8. Grid mode — bypassing the 120-result cap

Google Maps caps a single search at roughly 120 results. To get more, tile the area. `--grid-bbox minLat,minLon,maxLat,maxLon --grid-cell <km>` divides the bounding box into ~1 km cells (or whatever you set), and runs one search per cell with `--geo` set to that cell's center.

The math is in `grid/grid.go`. Latitude is constant — one degree of latitude is ~111.32 km everywhere — so the lat step is `cellSizeKm / 111.32`. Longitude shrinks toward the poles; we use the cosine of the midpoint latitude as the correction factor so the cells stay roughly square on the ground.

```bash
sed -n '94,120p' grid/grid.go
```

```output
// GenerateCells divides bbox into a grid where each cell is approximately
// cellSizeKm × cellSizeKm. It returns the center point of every cell.
//
// The longitude step is adjusted for the latitude of the bounding box centre
// so that cells are roughly square on the ground.
//
// Example: a 20×20 km area with cellSizeKm=1 produces ~400 cells.
func GenerateCells(bbox BoundingBox, cellSizeKm float64) []Cell {
	cellSizeKm = normalizeCellSizeKm(cellSizeKm)

	// Latitude step is constant everywhere.
	latStep := cellSizeKm / kmPerDegreeLat

	// Longitude step varies with latitude; use the midpoint for a good estimate.
	lonStep := calculateLonStep(bbox, cellSizeKm)

	var cells []Cell

	// Start at the centre of the first cell (half a step from the edge).
	for lat := bbox.MinLat + latStep/2; lat < bbox.MaxLat; lat += latStep {
		for lon := bbox.MinLon + lonStep/2; lon < bbox.MaxLon; lon += lonStep {
			cells = append(cells, Cell{Lat: lat, Lon: lon})
		}
	}

	return cells
}
```

`CreateGridSeedJobs` in `runner/jobs.go` then takes the Cartesian product of queries × cells and produces one `GmapJob` per (query, cell) pair. The shared deduper across all those jobs is what prevents the inevitable overlap between neighbouring cells from being scraped twice.

## 9. PostgreSQL job queue — `runner/databaserunner` + `postgres/`

The database runner is for distributed scraping: one instance with `-produce -dsn ...` writes seed jobs into Postgres and exits; many other instances with just `-dsn ...` pull jobs from the same database and process them. Workers compete for jobs using `SELECT ... FOR UPDATE SKIP LOCKED`, which gives lock-free job stealing.

```bash
sed -n '148,170p' postgres/provider.go
```

```output
func (p *provider) fetchJobs(ctx context.Context) {
	defer close(p.jobc)
	defer close(p.errc)

	q := `
	WITH updated AS (
		UPDATE gmaps_jobs
		SET status = $1
		WHERE id IN (
			SELECT id from gmaps_jobs
			WHERE status = $2
			ORDER BY priority ASC, created_at ASC FOR UPDATE SKIP LOCKED 
		LIMIT $3
		)
		RETURNING *
	)
	SELECT payload_type, payload from updated ORDER by priority ASC, created_at ASC
	`

	baseDelay := time.Millisecond * 50
	maxDelay := time.Millisecond * 300
	factor := 2
	currentDelay := baseDelay
```

Job payloads are encoded with `encoding/gob` and stored as raw bytes in the `payload` column with a string `payload_type` ("search" / "place" / "email") to drive decoding on the way out. The result writer in the same package batches up to 50 entries at a time into a single multi-row insert. Note the special handling of error code `22P05` ("unsupported Unicode escape sequence") — PostgreSQL JSONB rejects NUL bytes (`\x00`), so when a multi-row insert fails for that reason we fall back to one-by-one inserts after running `jsonbsanitize.StripNULFromEntry` on each.

## 10. The local web runner — `runner/webrunner/` + `web/`

Run the binary with no arguments and you get the web UI, listening on `:8080` by default. It is a self-contained tool intended for individuals: jobs are stored in an embedded SQLite database (`webdata/jobs.db`), CSVs are written to the same `webdata/` directory, and the same `Service` powers both the HTML form and the JSON REST API. The HTML templates and CSS are embedded into the binary via `//go:embed`.

The architecture is two goroutines tied together by an `errgroup`:

1. **HTTP server** — receives form posts and API requests, validates them, persists a `web.Job` row with `status='pending'`, and immediately returns the job to the user.
2. **Background worker** — every second, polls `SELECT * FROM jobs WHERE status='pending' ORDER BY created_at DESC LIMIT 1`, runs that job to completion using the same `CreateSeedJobs`/`scrapemateapp` pipeline as the file runner, and updates its status to `ok` or `failed`.

```bash
sed -n '67,82p' runner/webrunner/webrunner.go
```

```output
func (w *webrunner) Run(ctx context.Context) error {
	egroup, ctx := errgroup.WithContext(ctx)

	egroup.Go(func() error {
		return w.work(ctx)
	})

	egroup.Go(func() error {
		return w.srv.Start(ctx)
	})

	return egroup.Wait()
}

func (w *webrunner) Close(context.Context) error {
	return nil
```

The work loop is intentionally simple. Note that it runs jobs strictly sequentially — `MaxWorkers` of the queue is effectively 1. This is fine for the local UI because each job is itself parallelized internally by scrapemate (using `cfg.Concurrency` workers within a single scrape).

```bash
sed -n '85,131p' runner/webrunner/webrunner.go
```

```output
func (w *webrunner) work(ctx context.Context) error {
	ticker := time.NewTicker(time.Second)
	defer ticker.Stop()

	for {
		select {
		case <-ctx.Done():
			return nil
		case <-ticker.C:
			jobs, err := w.svc.SelectPending(ctx)
			if err != nil {
				return err
			}

			for i := range jobs {
				select {
				case <-ctx.Done():
					return nil
				default:
					t0 := time.Now().UTC()
					if err := w.scrapeJob(ctx, &jobs[i]); err != nil {
						params := map[string]any{
							"job_count": len(jobs[i].Data.Keywords),
							"duration":  time.Now().UTC().Sub(t0).String(),
							"error":     err.Error(),
						}

						evt := tlmt.NewEvent("web_runner", params)

						_ = runner.Telemetry().Send(ctx, evt)

						log.Printf("error scraping job %s: %v", jobs[i].ID, err)
					} else {
						params := map[string]any{
							"job_count": len(jobs[i].Data.Keywords),
							"duration":  time.Now().UTC().Sub(t0).String(),
						}

						_ = runner.Telemetry().Send(ctx, tlmt.NewEvent("web_runner", params))

						log.Printf("job %s scraped successfully", jobs[i].ID)
					}
				}
			}
		}
	}
}
```

### 10.1 The HTTP layer — `web/web.go`

The HTTP server uses the standard library's `net/http` mux with method-aware routes. There's an HTML form path (`/scrape`, `/jobs`, `/download`, `/delete`), an REST API path (`/api/v1/jobs`, `/api/v1/jobs/{id}`), and a Redoc-based documentation viewer at `/api/docs`. A `securityHeaders` middleware sets a strict CSP, anti-clickjacking and MIME-sniffing headers.

A nice detail: `requestWithID` parses the job UUID exactly once into the request context, and every handler that needs the ID calls `getIDFromRequest`. This validates the UUID immediately at the edge and centralizes the path-versus-query-string fallback in one place.

```bash
sed -n '186,205p' web/web.go
```

```output
func requestWithID(r *http.Request) *http.Request {
	id := r.PathValue("id")
	if id == "" {
		id = r.URL.Query().Get("id")
	}

	parsed, err := uuid.Parse(id)
	if err == nil {
		r = r.WithContext(context.WithValue(r.Context(), idCtxKey, parsed))
	}

	return r
}

func getIDFromRequest(r *http.Request) (uuid.UUID, bool) {
	id, ok := r.Context().Value(idCtxKey).(uuid.UUID)

	return id, ok
}

```

The `web.Job` struct in `web/job.go` is the domain object that both layers (HTML and JSON API) hand around. `Validate` ensures every required field is set; the SQLite repo in `web/sqlite/sqlite.go` only stores the `Data` field as JSON in a single column and reconstitutes it on read.

## 11. AWS Lambda runner — `runner/lambdaaws/`

There are two Lambda-related modes, paired by design: a *runner* that you deploy *as* a Lambda, and an *invoker* that runs locally and fires N Lambdas in parallel.

### 11.1 The runner — invoked when AWS hands it a chunk

Inside the Lambda runtime, the handler bootstraps Playwright (browsers and the Go driver are baked into the container image at `/opt/browsers` and `/opt/ms-playwright-go`, but they have to be copied to `/tmp` because that's the only writable directory in Lambda), runs the scrape via the same `CreateSeedJobs`/`scrapemateapp` pipeline, writes CSV to `/tmp/output.csv`, then uploads it to S3 keyed by `<jobID>-<part>.csv`.

```bash
sed -n '54,131p' runner/lambdaaws/lambdaaws.go
```

```output
func (l *lambdaAwsRunner) handler(ctx context.Context, input lInput) error {
	tmpDir := "/tmp"
	browsersDst := filepath.Join(tmpDir, "browsers")
	driverDst := filepath.Join(tmpDir, "ms-playwright-go")

	if err := l.setupBrowsersAndDriver(browsersDst, driverDst); err != nil {
		return err
	}

	out, err := os.Create(filepath.Join(tmpDir, "output.csv"))
	if err != nil {
		return err
	}

	defer out.Close()

	app, err := l.getApp(ctx, input, out)
	if err != nil {
		return err
	}

	in := strings.NewReader(strings.Join(input.Keywords, "\n"))

	var seedJobs []scrapemate.IJob

	exitMonitor := exiter.New()

	seedJobs, err = runner.CreateSeedJobs(
		false, // TODO supoort fast mode
		input.Language,
		in,
		input.Depth,
		false,
		"",
		0,
		10000, // TODO support radius
		nil,
		exitMonitor,
		input.ExtraReviews,
	)
	if err != nil {
		return err
	}

	exitMonitor.SetSeedCount(len(seedJobs))

	bCtx, cancel := context.WithTimeout(ctx, time.Minute*10)
	defer cancel()

	exitMonitor.SetCancelFunc(cancel)

	go exitMonitor.Run(bCtx)

	err = app.Start(bCtx, seedJobs...)
	if err != nil && !errors.Is(err, context.DeadlineExceeded) && !errors.Is(err, context.Canceled) {
		return err
	}

	out.Close()

	if l.uploader != nil {
		key := fmt.Sprintf("%s-%d.csv", input.JobID, input.Part)

		fd, err := os.Open(out.Name())
		if err != nil {
			return err
		}

		err = l.uploader.Upload(ctx, input.BucketName, key, fd)
		if err != nil {
			return err
		}
	} else {
		log.Println("no uploader set results are at ", out.Name())
	}

	return nil
}
```

### 11.2 The invoker — sharding work across many Lambdas

The invoker reads the local input file, chunks it into groups of `--aws-lambda-chunk-size` lines (default 100), and fires one Lambda per chunk via the AWS SDK. Each Lambda writes its output to `s3://<bucket>/<jobID>-<part>.csv`; you then download and concatenate them after the fact.

## 12. Output writers

Every runner builds one or more `scrapemate.ResultWriter`s. The same `*Entry` flows through them — only the destination changes.

| Writer | Built where | What it does |
|---|---|---|
| `csvwriter.NewCsvWriter` (from scrapemate) | `filerunner.setWriters` (default), `webrunner.setupMate`, `lambdaaws.getApp` | One CSV row per entry, well-known column order. |
| `jsonwriter.NewJSONWriter` (from scrapemate) | `filerunner.setWriters` when `--json` is set | Newline-delimited JSON (NDJSON). |
| Custom plugin | `filerunner.setWriters` when `--writer dir:Sym` is set | Loads any `*scrapemate.ResultWriter` symbol exported by a `.so`/`.dll`. |
| `leadsdb.New(apiKey)` | `filerunner.setWriters` when `--leadsdb-api-key` is set | Batches 100 entries (or 1 minute) into LeadsDB. |
| `postgres.NewResultWriter(conn)` | `databaserunner.New` | Multi-row `INSERT INTO results (data) VALUES ...` into Postgres with NUL-byte recovery. |
| `scraper.CentralWriter` | `scraper.NewScraperManager` (SaaS only) | Buffers in memory, flushes on exit-monitor signal, persists to `scrape_results` table. |

The LeadsDB writer is the easiest example to read end-to-end. It implements scrapemate's `ResultWriter` interface (a single `Run(ctx, in <-chan Result)` method), buffers up to 100 entries or 60 seconds, then calls the LeadsDB SDK's `BulkCreate`.

```bash
sed -n '37,74p' leadsdb/leadsdb.go
```

```output
func (l *leadsDBWriter) Run(ctx context.Context, in <-chan scrapemate.Result) error {
	const maxBatchSize = 100

	buff := make([]*leadsdb.Lead, 0, maxBatchSize)
	lastSave := time.Now().UTC()

	for result := range in {
		entry, ok := result.Data.(*gmaps.Entry)
		if !ok {
			return errors.New("invalid data type")
		}

		lead, err := convertToLead(entry)
		if err != nil {
			return err
		}

		buff = append(buff, lead)

		if len(buff) >= maxBatchSize || time.Now().UTC().Sub(lastSave) >= time.Minute {
			err := l.batchSave(ctx, buff)
			if err != nil {
				return err
			}

			buff = buff[:0]
		}
	}

	if len(buff) > 0 {
		err := l.batchSave(ctx, buff)
		if err != nil {
			return err
		}
	}

	return nil
}
```

## 13. The SaaS Edition — `cmd/gmapssaas/` and friends

The SaaS Edition is a separate binary that turns this scraper into a multi-user platform. It is gated entirely behind `cmd/gmapssaas/main.go` — none of this runs when you launch the main binary. It has five subcommands:

```bash
sed -n '22,55p' cmd/gmapssaas/main.go
```

```output
func main() {
	cmd := &cli.Command{
		Name:    "gmapssaas",
		Usage:   "Google Maps Scraper Pro",
		Version: "1.0.0",
		Flags: []cli.Flag{
			&cli.BoolFlag{
				Name:  "debug",
				Usage: "Enable debug logging",
			},
		},
		Before: func(ctx context.Context, cmd *cli.Command) (context.Context, error) {
			level := slog.LevelInfo
			if cmd.Bool("debug") {
				level = slog.LevelDebug
			}
			log.Init(level)
			return ctx, nil
		},
		Commands: []*cli.Command{
			cmdserve.Command,
			cmdworker.Command,
			cmdprovision.Command,
			cmdupdate.Command,
			cmdadmin.Command,
		},
	}

	if err := cmd.Run(context.Background(), os.Args); err != nil {
		log.Error("application failed", "error", err)
		os.Exit(1)
	}
}
```

The five subcommands form one logical system. `serve` runs the API + admin UI server (and the maintenance job queue). `worker` runs a scraper-worker process that pulls scrape jobs from the queue. `provision` is an interactive wizard that creates cloud VMs to run more workers. `admin` is a small set of CLI maintenance commands (e.g., create the initial admin user). `update` updates the binary in place. In production you typically run one `serve` instance and one-or-many `worker` instances against the same Postgres.

The architecture splits cleanly along three axes:

- **Persistence** is Postgres. All state — users, sessions, API keys, app config, encrypted secrets, queue jobs, and scrape results — lives in the same database. Migrations in `migrations/*.sql` are embedded into the binary and run on startup.
- **Job queue** is [River](https://riverqueue.com/), backed by the same Postgres. River has two queues: `default` (only the worker processes scrape jobs) and `maintenance` (only the server processes admin tasks like worker provisioning, worker health checks, and cleanup).
- **Two parallel HTTP routers** mounted on the same `chi.Router` in `cmd/gmapssaas/cmdserve/cmd_serve.go`: an `/api/v1/*` JSON API (in `api/`) and an admin UI (in `admin/`).

### 13.1 The server side — `cmdserve` and `rqueue`

When you run `gmapssaas serve`, the command in `cmd/gmapssaas/cmdserve/cmd_serve.go` wires everything together. It needs an `ENCRYPTION_KEY` (used by `cryptoext` to AES-GCM-encrypt secrets stored in `app_config` — provider tokens, the database URL, etc.), connects to Postgres, creates the River client *without* a `ScrapeWorker` (the server doesn't process scrapes, only maintenance), mounts the admin and API routers, and starts a Swagger handler for the API.

```bash
sed -n '146,175p' cmd/gmapssaas/cmdserve/cmd_serve.go
```

```output

		apiState := api.NewAppState(rqueueClient, apiStore)

		// Setup router
		mainRouter := chi.NewRouter()
		mainRouter.Use(middleware.Recoverer)

		// Setup admin routes
		admin.Routes(mainRouter, adminState, riverUIHandler)

		// Setup API routes (in a group so middleware can be added)
		mainRouter.Group(func(r chi.Router) {
			api.Routes(r, apiState)
		})

		// Swagger UI
		mainRouter.Get("/swagger/*", httpSwagger.Handler(
			httpSwagger.URL("/swagger/doc.json"),
		))

		srv, err := httpext.New(mainRouter, httpext.WithAddr(addr))
		if err != nil {
			return err
		}

		log.Info("starting server", "addr", addr)

		return srv.Run(ctx)
	},
}
```

The REST API is in `api/api.go`. It's a thin layer: every endpoint is authenticated with an API key (`KeyAuth` middleware looks up the hash in `api_keys` and stashes the key ID in the request context), and the handlers translate JSON into River job inserts via `appState.RQueue.InsertJob(...)`. `POST /api/v1/scrape` is the main one and returns immediately with a job ID; the caller polls `GET /api/v1/jobs/{id}` until done.

```bash
sed -n '38,52p' api/api.go
```

```output
// Routes sets up API routes on the given router.
func Routes(r chi.Router, appState *AppState) {
	r.Use(httpext.LoggingMiddleware)
	r.Use(middleware.Recoverer)
	r.Use(middleware.Timeout(120 * time.Second))
	r.Use(KeyAuth(appState.Store.ValidateAPIKey))

	r.Route("/api/v1", func(r chi.Router) {
		r.Get("/health", healthCheckHandler(appState))
		r.Post("/scrape", scrapeHandler(appState))
		r.Get("/jobs", listJobsHandler(appState))
		r.Get("/jobs/{job_id}", getJobHandler(appState))
		r.Delete("/jobs/{job_id}", deleteJobHandler(appState))
	})
}
```

### 13.2 The worker side — `cmdworker`, `scraper`, and `rqueue`

`gmapssaas worker` runs the scraper-side process. The interesting architectural choice here is that *one worker process runs exactly one River job at a time* (`MaxWorkers: 1` on the queue). Concurrency is achieved by running multiple worker processes (typically one per host), not by multiplexing inside a single process. This avoids contention on Playwright browser pages and lets the scraper-manager restart cleanly between jobs.

```bash
sed -n '466,492p' rqueue/rqueue.go
```

```output
func NewWorkerClient(dbPool *pgxpool.Pool, manager ScrapeManager) (*Client, error) {
	logger := log.With("component", "river-worker")

	workers := river.NewWorkers()
	river.AddWorker(workers, &ScrapeWorker{
		Manager: manager,
	})

	riverClient, err := river.NewClient(riverpgxv5.New(dbPool), &river.Config{
		Queues: map[string]river.QueueConfig{
			river.QueueDefault: {MaxWorkers: 1},
		},
		Workers:              workers,
		Logger:               logger,
		JobTimeout:           maxScrapeTimeout + 2*time.Minute,
		RescueStuckJobsAfter: 20 * time.Minute,
	})
	if err != nil {
		return nil, err
	}

	return &Client{
		riverClient: riverClient,
		dbPool:      dbPool,
	}, nil
}

```

The bridge between River and scrapemate is the `ScrapeManager` in `scraper/scraper.go`. River hands a `ScrapeJobArgs` to `ScrapeWorker.Work`, which calls into the manager to register the job, build the appropriate scrapemate job (`GmapJob` or `SearchJob`), submit it to the manager's internal channel-based provider (`scraper/provider.go`), and then *waits on a completion channel* that the central writer sends to.

`ScrapeManager.Run` is a restart loop. Every `--max-jobs-per-cycle` jobs (default 100), the entire scrapemate app is torn down and rebuilt. This is a heap-cleanup measure — Playwright leaks memory if it's left running indefinitely, and a clean restart between batches keeps the worker stable for long-running deployments.

```bash
sed -n '146,227p' scraper/scraper.go
```

```output
// Run starts the scraper manager loop. It creates a new scraper, runs it until
// the job threshold is reached, then restarts.
func (m *ScraperManager) Run(ctx context.Context) error {
	for {
		select {
		case <-ctx.Done():
			return nil
		default:
		}

		if err := m.runCycle(ctx); err != nil {
			if ctx.Err() != nil {
				return nil
			}

			return err
		}
	}
}

func (m *ScraperManager) runCycle(ctx context.Context) error {
	provider := NewProvider(m.concurrency * 64)

	// Update references atomically
	m.mu.Lock()
	m.provider = provider
	m.jobCount.Store(0)
	m.mu.Unlock()

	// Create scraper app
	app, err := m.createApp(provider)
	if err != nil {
		return err
	}

	defer func() { _ = app.Close() }()

	// Create cycle context
	cycleCtx, cycleCancel := context.WithCancel(ctx)
	defer cycleCancel()

	// Start scraper in goroutine with panic recovery
	scraperDone := make(chan error, 1)
	go func() {
		defer func() {
			if r := recover(); r != nil {
				scraperDone <- fmt.Errorf("scraper panic: %v", r)
			}
		}()
		scraperDone <- app.Start(cycleCtx)
	}()

	log.Info("scraper cycle started", "max_jobs", m.maxJobs)

	// Wait for restart signal, scraper done, or context cancelled
	select {
	case <-ctx.Done():
		cycleCancel()
		<-scraperDone

		return nil

	case err := <-scraperDone:
		// Scraper exited unexpectedly
		if ctx.Err() != nil {
			return nil
		}

		return err

	case <-m.restartChan:
		log.Info("restart triggered, restarting scraper",
			"jobs_processed", m.jobCount.Load(),
		)

		cycleCancel()
		<-scraperDone

		return nil
	}
}

```

### 13.3 `CentralWriter` — the writer that owns completion

The SaaS edition needs to know *exactly* when a single River job is done, including its results being safely persisted. The file-runner-style "wait for the exiter to signal done" isn't enough: we also need to commit results to Postgres before reporting success back to River, and we need to do it for one job at a time without interleaving rows from other in-flight jobs.

`CentralWriter` in `scraper/centralwriter.go` solves this. It satisfies scrapemate's `ResultWriter` interface, but instead of streaming results to a sink as they arrive, it accumulates them in memory under a `trackedJob` keyed by the River-job ID. When the exit monitor for that job fires (or when River's timeout fires), `Flush(jobID)` is called: it pulls the entries out, sanitizes NUL bytes, runs the `SaveFunc` (PostgreSQL multi-row insert by default), and sends a `FlushResult` on the completion channel that `ScrapeWorker.Work` is blocked on. That's how a "done" signal makes it all the way back to River so River can mark the job complete or schedule a retry.

```bash
sed -n '128,168p' scraper/centralwriter.go
```

```output
func (cw *CentralWriter) Flush(jobID string) {
	cw.mu.Lock()
	j := cw.current

	if j == nil || j.jobID != jobID {
		cw.mu.Unlock()
		return
	}

	cw.current = nil
	cw.mu.Unlock()

	for _, entry := range j.entries {
		jsonbsanitize.StripNULFromEntry(entry)
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	err := cw.save(ctx, j.riverJobID, j.keyword, j.entries)

	cancel()

	if err != nil {
		log.Error("failed to save results",
			"job_id", j.jobID,
			"river_job_id", j.riverJobID,
			"error", err,
		)
	} else if cw.OnResultsSaved != nil && len(j.entries) > 0 {
		cw.OnResultsSaved(len(j.entries))
	}

	j.completion <- FlushResult{ResultCount: len(j.entries), Err: err}

	log.Debug("flushed scrape job",
		"job_id", j.jobID,
		"river_job_id", j.riverJobID,
		"result_count", len(j.entries),
		"duration_ms", time.Since(j.startedAt).Milliseconds(),
		"save_error", err,
	)
}
```

This is also why the gmaps jobs have a `WriterManagedCompletion` flag. When set (only by `rqueue.ScrapeWorker.Work`), the jobs *do not* call `IncrPlacesCompleted` themselves on success; they let the central writer do it instead, from `markCompletedFromResult`. That avoids a race where the exiter declares "all places completed" before the writer has actually saved them.

### 13.4 The admin UI — `admin/`

`gmapssaas serve` also mounts an admin HTML UI at the root. It's a server-rendered html/template app — there's no SPA. `admin/routes.go` and `admin/templates/` make up the bulk of it. The features layer on top of the same Postgres tables: users, sessions, TOTP 2FA setup, API key management, queue inspection (via the embedded riverui handler), worker provisioning, and a terminal-over-WebSocket for SSH-ing into provisioned workers.

Most handlers follow the same pattern as `web/web.go`: parse the form, validate, persist, redirect or render a template. The interesting ones are `handlers_terminal.go` (WebSocket-based SSH terminal in the browser), `handlers_2fa.go` (TOTP enrolment with QR codes via `pquerna/otp` and `skip2/go-qrcode`), and `handlers_workers.go` (which inserts maintenance jobs into the River queue to provision/delete VMs on DigitalOcean or Hetzner).

### 13.5 Provisioning — `infra/` and `cmdprovision`

`gmapssaas provision` is an interactive CLI that walks a fresh installer through standing up the SaaS Edition on cloud infrastructure. The provider abstraction is in `infra/infra.go`:

```bash
sed -n '14,32p' infra/infra.go
```

```output
type Provisioner interface {
	// CheckConnectivity checks if the provision can access the provider
	CheckConnectivity(ctx context.Context) error
	// ExecuteCommand executes a command on the provider
	ExecuteCommand(ctx context.Context, command string) (string, error)
	// CreateDatabase provisions a database in the provier
	CreateDatabase(ctx context.Context) (*DatabaseInfo, error)
	// Deploy deploys the provided config on the provider
	Deploy(ctx context.Context, cfg *DeployConfig) error
}

// DeployConfig is a struct that holds all the required inforation a provider
// neds to deploy an image
type DeployConfig struct {
	Registry      *RegistryConfig
	DatabaseURL   string
	EncryptionKey string
	HashSalt      string
}
```

Two providers ship in-tree: DigitalOcean (`infra/digitalocean`) and Hetzner Cloud (`infra/hetzner`). The provisioning wizard's state is persisted between runs in `~/.gmapssaas/state.json` so you can resume after, e.g., your SSH session drops.

After the database is up, workers are added via the admin UI: it inserts a `WorkerProvisionArgs` job into the River `maintenance` queue, which the server-side River client picks up and processes by calling the provider's `Deploy(...)`. The worker VM is configured (via `infra/cloudinit/`) to start the `gmapssaas worker` container on boot, pointing at the central Postgres. The worker then connects, starts pulling from the `default` River queue, and reports its health every 30 seconds.

## 14. Telemetry — `tlmt/`

`runner.Telemetry()` is a lazy singleton. By default it instantiates a PostHog client (`tlmt/goposthog`) keyed by a hashed-IP-and-OS machine ID. Set `DISABLE_TELEMETRY=1` and you get the no-op telemetry from `tlmt/gonoop` instead.

```bash
sed -n '54,84p' tlmt/tlmt.go
```

```output
func generateMachineID() machineIdentifier {
	once.Do(func() {
		ip := fetchExternalIP()
		if ip == "" {
			ip = uuid.New().String()
		}

		hash := sha256.New()
		hash.Write([]byte(ip))
		hash.Write([]byte(runtime.GOARCH))
		hash.Write([]byte(runtime.GOOS))
		hash.Write([]byte(runtime.Version()))

		id := fmt.Sprintf("%x", hash.Sum(nil))

		meta := make(map[string]any)

		info, err := host.Info()
		if err == nil {
			meta["os"] = info.OS
			meta["platform"] = info.Platform
			meta["platform_family"] = info.PlatformFamily
			meta["platform_version"] = info.PlatformVersion
		}

		identifier.id = id
		identifier.meta = meta
	})

	return identifier
}
```

The events that are sent are the high-level facts of a run (job count, duration, error if any). Each runner has a `tlmt.NewEvent("...")` call near its `Run` entry point. The author uses this to understand which run modes people use.

## 15. Building and packaging

The Makefile in the repo root has all of the build targets you'll typically use. The `Dockerfile` is multi-stage: stage 1 downloads the Playwright browser binaries; stage 2 builds the Go binary; stage 3 is a slim Debian image with just the runtime dependencies needed by Chromium plus the binaries copied from stages 1 and 2. There's also a `Dockerfile.saas` for the SaaS edition.

```bash
grep -E '^[a-z][a-zA-Z_-]*:.*##' Makefile | head -20
```

```output
help: ## help information about make commands
vet: ## runs go vet
format: ## runs go fmt
test: ## runs the unit tests
test-cover: ## outputs the coverage statistics
test-cover-report: ## an html report of the coverage statistics
vuln: ## runs vulnerability checks
lint: ## runs the linter
cross-compile: ## cross compiles the application
build: ## builds the application (default: playwright)
docker: ## builds docker image with playwright (default)
build-saas: ## builds the SaaS binary (API server, worker, admin)
docker-saas: ## builds docker image for SaaS
saas-docker-push: docker-saas ## builds and pushes the SaaS docker image
provision: ## run the provisioning wizard via Docker (state persisted to ~/.gmapssaas)
saas-dev: ## start SaaS development environment (postgres + migrations + admin user + hot reload)
saas-dev-stop: ## stop SaaS development environment
saas-dev-reset: ## reset SaaS development environment (drops all data)
saas-run-server: ## run the SaaS API server locally
saas-run-worker: ## run the SaaS worker locally
```

The `cross-compile` target produces three binaries (linux/darwin/windows, amd64) at the configured `VERSION`. CI in `.github/workflows/build.yml` uses these.

## 16. Tests

Most testing happens at the unit level. A representative example is `gmaps/entry_test.go`, which feeds raw `APP_INITIALIZATION_STATE` payloads (captured in `testdata/`) through `EntryFromJSON` and asserts the resulting struct — this is where regressions in Google's HTML/JSON layout get caught. Other notable test files: `grid/grid_test.go` (cell-generation math), `scraper/centralwriter_test.go` (Flush sequencing), `rqueue/rqueue_test.go` (worker behaviour with a fake manager), and `runner/jobs_test.go` (input-line parsing).

Run them with:

```bash
find . -name '*_test.go' -not -path './.git/*' | sort
```

```output
./cmd/gmapssaas/cmdupdate/cmd_update_test.go
./gmaps/entry_test.go
./gmaps/reviews_test.go
./grid/grid_test.go
./infra/cloudinit/cloudinit_test.go
./infra/vps/provisioner_test.go
./internal/jsonbsanitize/entry_test.go
./rqueue/rqueue_test.go
./runner/jobs_test.go
./scraper/centralwriter_test.go
./scraper/provider_test.go
./scraper/scraper_test.go
```

## 17. Putting it all together — a real request, end to end

To pin everything down, let's walk through what happens when a user runs:

> `echo "coffee shops in Berlin" | ./google-maps-scraper -input /dev/stdin -results out.csv -email`

1. `main.main` initialises signal handling, calls `runner.ParseConfig`.
2. `ParseConfig` sees `-input` set, `-dsn` empty, `-web` not set → falls through to the `cfg.Dsn == ""` arm of the switch → `cfg.RunMode = RunModeFile`.
3. `main.runnerFactory(cfg)` returns a `filerunner.fileRunner` whose `input` is stdin, whose writer is a CSV writer to `out.csv`, and whose scrapemate app is configured with `WithJS(DisableImages())` (we're not in fast mode, not in debug).
4. `runnerInstance.Run(ctx)` is called. `fileRunner.Run` builds a fresh deduper and exiter, reads stdin, and produces one `*gmaps.GmapJob` for the line "coffee shops in Berlin" with `ExtractEmail: true`, deduper and exiter attached.
5. `exitMonitor.SetSeedCount(1)`. A goroutine starts `exitMonitor.Run(ctx)` — it'll block until done.
6. `app.Start(ctx, seedJobs...)` enters scrapemate's main loop. A browser worker dequeues the seed `GmapJob`, opens a Chromium page, calls `GmapJob.BrowserActions` which scrolls the side feed up to `MaxDepth` times. The rendered HTML comes back. scrapemate's HTTP fetcher parses it into a `goquery.Document` and calls `GmapJob.Process`.
7. `Process` finds 20-ish anchors in `div[role=feed] div[jsaction]>a`, deduplicates their hrefs through the shared deduper, and returns 20 `*gmaps.PlaceJob` children. It also calls `exitMonitor.IncrPlacesFound(20)` and `IncrSeedCompleted(1)`. The exiter now knows "1/1 seeds done, 0/20 places done".
8. scrapemate enqueues those 20 `PlaceJob`s. The browser pool picks them up in parallel (concurrency = `cfg.Concurrency`, default ½ CPU). For each one, `PlaceJob.BrowserActions` opens the place URL, polls `window.APP_INITIALIZATION_STATE` for the JSON blob, and returns it in `resp.Meta["json"]`.
9. `PlaceJob.Process` runs `EntryFromJSON` to get an `*Entry`. Because `-email` was set and the entry has a `WebSite`, it sets `j.UsageInResultststs = false`, builds an `EmailExtractJob` carrying the entry by pointer, and returns the email job as a child.
10. The email job is plain HTTP. It visits `entry.WebSite`, scans for `mailto:` links and email-regex matches, fills `entry.Emails`, and returns the entry as its result. The CSV writer (registered with scrapemate) writes one row.
11. `EmailExtractJob.Process`'s deferred function calls `exitMonitor.IncrPlacesCompleted(1)`. After this fires for all 20 places, the exiter's `done` channel sends a signal, `cancelFunc()` is called, scrapemate's `app.Start` returns `context.Canceled`, `fileRunner.Run` returns that error, `main` sees it's `context.Canceled` (allowed!), the runner is closed, telemetry is flushed, and the process exits 0.

That single command exercises every layer of the engine except fast mode, grid mode, and the central writer. Substituting:

- `-web` would route into `webrunner` and the same scrapemate pipeline runs behind an HTTP form.
- `-dsn postgres://...` would route into `databaserunner`, and the `postgres.provider`/`postgres.NewResultWriter` pair would take the place of stdin and the CSV writer.
- `-fast-mode -geo 52.52,13.40 -zoom 14` (note: requires geo coordinates) would route through `SearchJob` instead of `GmapJob` and skip the browser entirely.
- `-grid-bbox 52.45,13.30,52.55,13.50 -grid-cell 1.0` would multiply step 4 by 100ish cells and rely on the shared deduper to suppress overlap between neighbouring searches.

In the SaaS Edition, the analogous flow is: a user POSTs to `/api/v1/scrape`, the API handler inserts a `ScrapeJobArgs` River job into the `default` queue, a `worker` process's `ScrapeWorker.Work` registers the job with the central writer, builds the gmaps job, submits it to the in-memory provider, the scraper-manager's scrapemate app dequeues and processes it exactly like steps 6-10 above, the central writer collects entries instead of streaming them to a CSV writer, and when the exit monitor fires the flush goroutine writes them to `scrape_results` as a JSONB blob and sends a `FlushResult` back to `Work`, which returns success to River, which marks the job completed.

That's the whole project.

