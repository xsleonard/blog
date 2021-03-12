+++
title = "Common setup for a public HTTP server in Go"
tags = ["software", "go", "golang", "http"]
date = "2019-10-28T00:00:00+00:00"
+++

These are common initial setups I use for public Go HTTP services.

<!-- MarkdownTOC -->

- [Configure the HTTP server with timeouts](#configure-the-http-server-with-timeouts)
- [Process lifecycle: graceful shutdown](#process-lifecycle-graceful-shutdown)
- [CORS](#cors)
- [unrolled/secure: Security headers, SSL redirect, host whitelisting](#unrolledsecure-security-headers-ssl-redirect-host-whitelisting)
- [Gzip](#gzip)
- [Let's Encrypt](#lets-encrypt)
- [Testing](#testing)

<!-- /MarkdownTOC -->


## Configure the HTTP server with timeouts

The default HTTP server in the [http package](https://golang.org/pkg/net/http/) is not suitable for use as a public Go HTTP server because it does not have timeouts configured. Inevitably, this default server will see connection exhaustion running as a public service.

The timeout configuration is explained well by this post:
https://blog.cloudflare.com/the-complete-guide-to-golang-net-http-timeouts/

I configure my default HTTP server with these timeouts, which are somewhat arbitrary. They should be reconsidered depending upon the server's use cases, but are suitable for a typical REST API.

```go
const (
    httpAddr := ":8080"
    serverReadTimeout  := time.Second * 10
    serverWriteTimeout := time.Second * 60
    serverIdleTimeout  := time.Second * 120
)

func newHTTPServer(handler http.Handler) *http.Server {
    return &http.Server{
        Addr:         httpAddr,
        Handler:      handler,
        ReadTimeout:  serverReadTimeout,
        WriteTimeout: serverWriteTimeout,
        IdleTimeout:  serverIdleTimeout,
    }
}
```

## Process lifecycle: graceful shutdown

The main process should be setup in a way that allows graceful shutdown
in the event of an error during startup of any goroutines, or if terminated
by SIGINT (ctrl+c).

This allows the server to close open connections and finish existing requests by using the [`http.Server.Shutdown`](https://golang.org/pkg/net/http/#Server.Shutdown) method.

```go
func run() error {
    // Setup HTTP server (using previous example code)
    handler := api.NewHandler() // Whatever handler is defined by your app
    httpServer := newHTTPServer(handler)

    // Create a WaitGroup to synchronize termination of all goroutines
    var wg sync.WaitGroup
    // Create an error channel to handle goroutine errors
    // The buffer size should equal the number of goroutines that are run
    errs := make(chan error, 2)

    // Run the HTTP server
    wg.Add(1)
    go func() {
        defer wg.Done()

        err := httpServer.ListenAndServe()
        if err != http.ErrServerClosed {
            errs <- err
        }
    }()

    // (Example) Run another app goroutine
    wg.Add(1)
    go func() {
        defer wg.Done()
        if err := app.Run(); err != nil {
            errs <- err
        }
    }

    // Catch ctrl+c and shutdown the server gracefully
    quit := make(chan struct{})
    go func() {
        c := make(chan os.Signal, 1)
        signal.Notify(c, syscall.SIGINT)
        <-c
        close(quit)
    }()

    // Wait for ctrl+c or for one of the goroutines to fail
    var err error
    select {
    case <-quit:
    case err = <-errs:
        log.Println("Goroutine failed:", err)
    }

    // Shutdown the HTTP server with a timeout
    // Note: the HTTP server may not be running if it failed to start
    serverShutdownTimeout := time.Second * 5
    ctx, cancel := context.WithTimeout(ctx, serverShutdownTimeout)
    defer cancel()
    if err := httpServer.Shutdown(ctx); err != nil {
        log.Println("shutdownServer error:", err)
    }

    log.Println("Waiting for goroutines to finish")
    wg.Wait()

    return err
}
```

## CORS

[CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) configuration is not needed if the application does not need to support cross-origin requests. By default, browsers block any cross-origin requests without any further configuration from the server. Only use CORS to allow other domains (origins) to make requests to your service. This includes subdomains, so this is commonly needed.

I use [rs/cors](https://github.com/rs/cors) for CORS configuration. Configuration is straightforward from their documentation: https://github.com/rs/cors#parameters

## unrolled/secure: Security headers, SSL redirect, host whitelisting

I use [unrolled/secure](https://github.com/unrolled/secure) to add various security headers such as `Content-Security-Policy`, to perform automatic SSL redirects or to restrict requests to certain hostnames.

## Gzip

If the HTTP service is not deployed behind something like nginx, I use the [nytimes/gziphandler](https://github.com/NYTimes/gziphandler) middleware for gzipping HTTP requests. Otherwise, this can be handled by nginx or a similar tool in a reverse proxy setup.

## Let's Encrypt

Free SSL certs with automatic renewal can be obtained from [Let's Encrypt](https://letsencrypt.org) using [x/crypto/acme/autocert](https://godoc.org/golang.org/x/crypto/acme/autocert). See the [certificate manager example](https://godoc.org/golang.org/x/crypto/acme/autocert#ex-Manager) for configuring an `http.Server` with autocert.

## Testing

HTTP handler tests follow this pattern, using [table-driven tests](https://github.com/golang/go/wiki/TableDrivenTests):

```go

import (
    "net/http"
    "net/http/httptest"
    "testing"

    "github.com/stretchr/testify/require"
)

func TestHandler(t *testing.T) {
    case := []struct{
        name string
        method string
        status int
        reqBody string
        resp string
    }{
        {
            name: "OK",
            method: http.MethodGet,
            status: http.StatusOK,
            reqBody: `{"id":123}`,
            resp: `{"id":123,"name":"foo"}\n`,
        },
        {
            name: "Method Not Allowed",
            method: http.MethodPost,
            status: http.StatusMethodNotAllowed,
            reqBody: `{"id":123}`,
            resp: "405 - Method Not Allowed\n",
        },
    }

    for _, tc := range cases {
        t.Run(tc.name, func(t *testing.T) {
            // Setup the http request
            endpoint := "/api/v1/thing"
            req, err := http.NewRequest(tc.method, endpoint, strings.NewReader(tc.reqBody))
            require.NoError(t, err)
            req.Header.Set("Content-Type", "application/json")

            // Pass the request through the HTTP server handler,
            // and record the result with httptest.ResponseRecorder
            rr := httptest.NewRecorder()
            handler := newHandler()
            handler.ServeHTTP(rr, req)

            // Check the response status code and body
            resp := w.Result()
            defer resp.Body.Close()

            require.Equal(t, tc.status, resp.StatusCode)

            body, err := ioutil.ReadAll(resp.Body)
            require.NoError(t, err)

            require.Equal(t, tc.resp, body)
        })
    }
}
```
