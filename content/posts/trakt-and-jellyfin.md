+++
date = '2025-11-27T17:42:03+01:00'
draft = false
title = 'trakt.tv and Jellyfin: when tracking your shows goes wrong'
+++

I run my own [Jellyfin](https://jellyfin.org/) server, as well as having a local OpenELEC (Kodi) for when I'm home, and use trakt.tv to keep my views in sync.

While Kodi does a pretty good job at picking up when something is marked as watched on Trakt, Jellyfin does not, so I sometimes watch a show at home, and some time later I'm somewhere else wanting to watch the next episode on my phone, and Jellyfin will still be stuck at those "older" episodes I already watched at home.

Jellyfin's interface is already somewhat annoying at this (I can't figure out, on Android, how to multi-select and then mark as watched) but it also has two very irritating behaviours:

1. When marking an episode as watched (as opposed to scrobbling it) Jellyfin will mark it as watched on Trakt.
2. Sometimes (and I can't figure out why or how) I tap _something_ and it marks the whole show as watched.

Both of these require me to go to trakt.tv and "remove this play" for every episode, which can get REALLY boring to do if you watched multiple episodes, and becomes downright horrible to do when you accidentally marked an _old-timey_ (i.e. 20+ episodes per season, yes I grew up on those), multi-season show as watched, but you're still on Season 1.

Thankfully, though, [Trakt has an API](https://trakt.docs.apiary.io/), and I have a passable understanding of Golang development, so I'm sure I can sort something out.

## But how?

This was actually fairly easy to figure out:

* The API uses OAuth for authentication, and [anyone can create an app](https://trakt.docs.apiary.io/#introduction/create-an-app).
* Their OAuth has a "devices" mode meant for non-browser based devices, of which CLI scripts are one
  * tl;dr: your app calls Trakt and gets a pairing code, shows it to you, you go to Trakt, login and type the code, confirm the connection, get an `access_token` and a `refresh_token` back.
* The API revolves around internal (Trakt) IDs
* You can use the [Sync](https://trakt.docs.apiary.io/#reference/sync) endpoint to:
  * Get your History
  * Remove specific items from the history

So here's the plan[^dang]:

1. Get the Trakt ID of the show I want to "clean up"
2. Get all the History entries of that show in my profile
3. Wipe them

Easy, right[^gg]?

## This is how

If I ever end up cleaning up my code (i.e. remove the stuff I hardcoded to make my life easier) I'll put it in a repo, for now it'll be a bunch of code blocks in here.

### Step 0: someone else's (awesome) library

I am extremely thankful to Florian Schwab of <https://gitlab.com/ydkn/go-trakt> fame for doing most of my work for me.

The only thing that's not in this library is getting the "code" to type into Trakt for authorization; that required a little bit of work on my part, and I ended up not using the library's `OAuthService` at all but just DIYing it with HTTP calls.

The other functions were already there and nicely laid out for me to use.

### Step 1: authorization

This was by far the most complex part (which tells you something).

First I read a configuration file (`.toml` in this case) and Marshal it directly into the library's `trakt.ClientConfig`. This struct is meant to hold the OAuth ID and secret, and the access token.

```go
// ClientConfig contains auth and connection details for the Trakt.tv API.
type ClientConfig struct {
    ClientID     string
    ClientSecret string
    AccessToken  string
}
```

if my config has all three then I consider myself authenticated and skip to the next part, otherwise I call this beauty:

```go
type CodeResponse struct {
    DeviceCode      string `json:"device_code"`
    UserCode        string `json:"user_code"`
    VerificationURL string `json:"verification_url"`
    ExpiresIn       int    `json:"expires_in"`
    Interval        int    `json:"interval"`
}

type Tokens struct {
    AccessToken  string `json:"access_token"`
    RefreshToken string `json:"refresh_token"`
    ExpiresIn    int64  `json:"expires_in"`
    Scope        string `json:"scope"`
    CreatedAt    int64  `json:"created_at"`
}

func Authenticate(apiBase string, clientConfig trakt.ClientConfig) (*Tokens, error) {
    // https://trakt.docs.apiary.io/#reference/authentication-devices
    // Generate codes. Your app calls /oauth/device/code to generate new codes. Save this entire response for later use.
    apiBase = strings.TrimSuffix(apiBase, "/") // Remove trailing slash
    cv := url.Values{}
    cv.Set("client_id", clientConfig.ClientID)
    resp, err := http.PostForm(fmt.Sprintf("%s/oauth/device/code", apiBase), cv)
    if err != nil {
        return nil, fmt.Errorf("error getting OAuth code from %q: %v", apiBase, err)
    }
    if resp.StatusCode != http.StatusOK {
        return nil, fmt.Errorf("error getting OAuth code from %q: got status %d", apiBase, resp.StatusCode)
    }
    defer resp.Body.Close()

    bb, err := io.ReadAll(resp.Body)
    if err != nil {
        return nil, fmt.Errorf("io.ReadAll: %v", err)
    }

    cr := CodeResponse{}
    if err := json.Unmarshal(bb, &cr); err != nil {
        return nil, fmt.Errorf("json.Unmarshal: %v", err)
    }
    // Display the code. Display the user_code and instruct the user to visit the verification_url on their computer or mobile device.

    tk, err := validateCode(apiBase, cr, clientConfig)
    if err != nil {
        return nil, fmt.Errorf("validateCode: %v", err)
    }

    return &tk, nil
}
```

we call the `/oauth/device/code` endpoint to get a code, which we marshal from JSON into a `CodeResponse`, which gives us:

* the 8 character code for verification
* the Verification URL (usually `https://trakt.tv/activate/`) where we will type the code
* how long until the code expires (600 seconds) and how often we should poll Trakt to see if the user did something (5 seconds)
* a "device code" that identifies our application when we poll (ties the code to the app, but it's a lot longer)

`apiBase` is `https://api.trakt.tv`; it's set like this so I can write tests for it, which I'll definitely do later.

The call to `validateCode` does exactly what you'd expect: polls Trakt at the `Interval` until it expires. This is a lovely use of the `retry` [Go library](https://github.com/sethvargo/go-retry) and its `Constant` backoff strategy. No `for` loops and `time.Sleep` needed.

```go
// Returns an access_token and a refresh token when the code validates.
func validateCode(apiBase string, cr CodeResponse, cc trakt.ClientConfig) (Tokens, error) {
    // Poll for authorization. Poll the /oauth/device/token method to see if the user successfully authorizes your app.
    // Use the device_code and poll at the interval (in seconds) to check if the user has authorized your app.
    // Check the docs below for the specific error codes you need to handle.
    // Use expires_in to stop polling after that many seconds, and gracefully instruct the user to restart the process.
    // It is important to poll at the correct interval and also stop polling when expired.
    ctx := context.Background()
    b := retry.NewConstant(time.Duration(cr.Interval) * time.Second)
    b = retry.WithMaxDuration(time.Duration(cr.ExpiresIn)*time.Second, b)
    tk := Tokens{}
    if err := retry.Do(ctx, b, func(_ context.Context) error {
        fmt.Printf("Please visit %q and insert code %q\n", cr.VerificationURL, cr.UserCode)
        apiBase = strings.TrimSuffix(apiBase, "/") // Remove trailing slash.
        cv := url.Values{}
        cv.Set("client_id", cc.ClientID)
        cv.Set("client_secret", cc.ClientSecret)
        cv.Set("code", cr.DeviceCode)
        resp, err := http.PostForm(fmt.Sprintf("%s/oauth/device/token", apiBase), cv)
        if err != nil {
            return fmt.Errorf("error getting OAuth code from %q: %v", apiBase, err)
        }
        if resp.StatusCode != http.StatusOK {
            // https://trakt.docs.apiary.io/#reference/authentication-devices/get-token
            switch resp.StatusCode {
            case 400:
                return retry.RetryableError(fmt.Errorf("Waiting for user to authorize."))
            case 404:
                return fmt.Errorf("invalid device_code")
            case 409:
                return fmt.Errorf("Already Used - user already approved this code")
            case 410:
                return fmt.Errorf("Expired - the tokens have expired, restart the process")
            case 418:
                return fmt.Errorf("Denied - user explicitly denied this code")
            case 429:
                return retry.RetryableError(fmt.Errorf("Slow Down - your app is polling too quickly"))
            }
            return fmt.Errorf("got unknown status code %d", resp.StatusCode)
        }
        defer resp.Body.Close()

        bb, err := io.ReadAll(resp.Body)
        if err != nil {
            return fmt.Errorf("io.ReadAll: %v", err)
        }

        if err := json.Unmarshal(bb, &tk); err != nil {
            return fmt.Errorf("json.Unmarshal: %v", err)
        }
        return nil
    }); err != nil {
        return Tokens{}, fmt.Errorf("retry: %v", err)
    }

    return tk, nil
}
```

Two errors will cause the library to try again (`RetryableError`): if we're going too fast, or if the user hasn't accepted or rejected yet. Everything else will cause an abort.

Once we succeed, we Marshal the JSON response in the `Tokens` struct, which contains the `AccessCode` that is missing from `trakt.ClientConfig`.

Authentication is done, we can now create a `trakt.NewClient(cfg)` where `cfg` is our OAuth client ID and secret, and the access token.

Since this is a "one-off" type of scenario, if we save the access token[^lttr] and put it in the config file we'll be authenticated for a while, so while we did get an OAuth refresh token that we could use, we just... Forget about it.

### Find the show ID

Somewhat surprisingly, trakt.tv (the web UI) doesn't show the show ID anywhere (I didn't dig into the page source code, maybe it's there?); this means we need to do a search and return matching shows and their IDs.

The `go-trakt` library uses individual clients for each section of the API, but all of them have at their "root" the `Client` from above.

We have our own custom struct

```go
type API struct {
    c *trakt.Client
}

func NewAPI(c *trakt.Client) API {
    return API{
        c: c,
    }
}
```

and we can plug functions into it, like

```go
// Search searches for a show by name.
func (a *API) SearchShow(ctx context.Context, text string) ([]*trakt.Show, error) {
    ss := a.c.Search()

    q, err := ss.TextQuery(ctx, trakt.ItemTypeShow, text)
    if err != nil {
        return nil, fmt.Errorf("TextQuery(): %v", err)
    }

    shows := []*trakt.Show{}

    for _, r := range q {
        shows = append(shows, &r.Show)
    }
    return shows, nil
}
```

and in our `main` all we need to do is print them out

```go
func searchShow(ctx context.Context, a api.API, text string) error {
    r, err := a.SearchShow(ctx, text)
    if err != nil {
        return fmt.Errorf("SearchShow(%q): %v", text, err)
    }

    fmt.Printf("Trakt ID\t Show\n")
    for _, s := range r {
        fmt.Printf("%d\t %s\n", s.IDs.Trakt, s.Title)
    }
    return nil
}
```

which would result into something like this

```console
$ go run main.go -action search -text "Marvel's Agents of S.H.I.E.L.D."
Trakt ID   Show
1394       Marvel's Agents of S.H.I.E.L.D.
152103     Marvel's Agents of S.H.I.E.L.D.: Slingshot
```

We now know that our show ID is `1394`

### When did I watch this again?

We pretty much do the same thing, except that we use the `Sync` client, call the `GetHistory` function and pass it the show ID:

```go
// GetHistory gets the view history of a show.
func (a *API) GetHistory(ctx context.Context, showID uint64) ([]*trakt.HistoryItem, error) {
    hs := a.c.Sync()
    h, err := hs.GetHistory(ctx, trakt.ItemTypeShow, showID)
    if err != nil {
        return nil, fmt.Errorf("GetHistory(%d): %v", showID, err)
    }
    return h, nil
}
```

We do a pretty print of the history and get something like this

```text
ID           Watched At                     Show Title, Season, and Episode
10000000027  2025-11-20 10:11:12 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S03E16
10000000026  2025-11-19 22:20:12 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S03E15
10000000025  2025-11-19 21:15:15 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S03E14
10000000024  2025-11-19 20:18:12 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S03E13
10000000023  2025-11-18 19:40:54 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S03E13
10000000022  2025-11-17 18:03:06 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S03E12
10000000021  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E10
10000000020  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E09
10000000019  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E08
10000000018  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E07
10000000017  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E06
10000000016  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E05
10000000015  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E04
10000000014  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E03
10000000013  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E02
10000000012  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S07E01
10000000011  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E13
10000000010  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E12
10000000009  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E11
10000000008  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E10
10000000007  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E09
10000000006  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E08
10000000005  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E07
10000000004  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E06
10000000003  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E05
10000000002  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E04
10000000001  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E03
10000000000  2025-11-17 01:02:03 +0000 UTC  Marvel's Agents of S.H.I.E.L.D. S06E02
```

Hmmm, curious: a ton of entries of increasing ID and Season/Episode, all "Watched At" the same exact time. I wonder what happened there...

### Get rid of them please

With the help of Microsoft Excel we can remove all the ones we want to keep from this list, and build a list of the IDs of the ones we want to get rid of

```text
10000000021,10000000020,10000000019,10000000018,10000000017,10000000016,10000000015,10000000014,10000000013,10000000012,10000000011,10000000010,10000000009,10000000008,10000000007,10000000006,10000000005,10000000004,10000000003,10000000002,10000000001,10000000000
```

The function call to remove the history looks pretty familiar:

```go
// RemoveFromHistory removes a specific view from the history
func (a *API) RemoveFromHistory(ctx context.Context, viewID uint64) error {
    hs := a.c.Sync()
    ok, err := hs.RemoveHistory(ctx, viewID)
    if err != nil {
        return fmt.Errorf("RemoveHistory(%d): %v", viewID, err)
    }
    if !ok {
        return fmt.Errorf("RemoveHistory(%d) returned false but no error details", viewID)
    }
    return nil
}
```

the only caveats are that:

1. we can only pass one entry at a time (and so we may need to look out for HTTP 429 errors)
2. we want to make sure that, especially on large numbers of deletions, we give the user a chance to abort before we start deleting stuff.

So we can do something like this:

```go
func forgetHistory(ctx context.Context, a api.API, historyIDsCSV string) error {
    ids := map[uint64]bool{}
    for _, id := range strings.Split(historyIDsCSV, ",") {
        v, err := strconv.ParseUint(strings.TrimSpace(id), 10, 0)

        if err != nil {
            return fmt.Errorf("invalid value %q in history_ids", strings.TrimSpace(id))
        }
        ids[v] = true
    }

    reader := bufio.NewReader(os.Stdin)
    fmt.Printf("WARNING!!! Are you sure you want to remove %d items from history (y/N)? ", len(ids))
    text, _ := reader.ReadString('\n')
    if strings.ToLower(text) != "y\n" { // The \n is needed or it won't match.
        return fmt.Errorf("User aborted")
    }
    for hi := range ids {
        log.Printf("Removing history item %d\n", hi)
        if err := a.RemoveFromHistory(ctx, hi); err != nil {
            return fmt.Errorf("removing item %d failed: %v", hi, err)
        }
    }

    log.Printf("%d items removed", len(ids))
    return nil
}
```

Using a `map[uint64]bool` to collect the IDs so we don't have to worry about duplicates that may have snuck in.

One quick-ish run later

```console
$ go run main.go -action forget -history_ids "10000000021,10000000020,10000000019,10000000018,10000000017,10000000016,10000000015,10000000014,10000000013,10000000012,10000000011,10000000010,10000000009,10000000008,10000000007,10000000006,10000000005,10000000004,10000000003,10000000002,10000000001,10000000000"
WARNING!!! Are you sure you want to remove 22 items from history (y/N)? y
2025/11/21 00:01:01 Removing history item 10000000000
2025/11/21 00:01:02 Removing history item 10000000021
2025/11/21 00:01:03 Removing history item 10000000020
2025/11/21 00:01:04 Removing history item 10000000019
2025/11/21 00:01:05 Removing history item 10000000018
2025/11/21 00:01:06 Removing history item 10000000017
2025/11/21 00:01:07 Removing history item 10000000016
2025/11/21 00:01:08 Removing history item 10000000015
2025/11/21 00:01:09 Removing history item 10000000014
2025/11/21 00:01:10 Removing history item 10000000013
2025/11/21 00:01:11 Removing history item 10000000012
2025/11/21 00:01:12 Removing history item 10000000011
2025/11/21 00:01:13 Removing history item 10000000010
2025/11/21 00:01:14 Removing history item 10000000009
2025/11/21 00:01:15 Removing history item 10000000008
2025/11/21 00:01:16 Removing history item 10000000007
2025/11/21 00:01:17 Removing history item 10000000006
2025/11/21 00:01:18 Removing history item 10000000005
2025/11/21 00:01:19 Removing history item 10000000004
2025/11/21 00:01:20 Removing history item 10000000003
2025/11/21 00:01:21 Removing history item 10000000002
2025/11/21 00:01:22 Removing history item 10000000001
2025/11/21 00:01:23 Removing history item 10000000002
2025/11/21 00:01:24 26 items removed
2025/11/21 00:02:24 Done!
```

And we're done :)

[^dang]: which I came up with after I figured out that among the various tools that use the Trakt API I could find, none of them solved this problem.
[^gg]: actually, yes.
[^lttr]: an exercise that is left to the reader.
