
# 🧾 CSRF Forbidden Origin Error – Debugging Summary

## 🧩 Problem

While implementing CSRF protection using [`gorilla/csrf`](https://github.com/gorilla/csrf) in a Go backend server, all POST requests to `/signup/users` were being rejected with a **403 Forbidden - Invalid Origin** status. Despite correctly sending both the CSRF cookie and form token, the middleware was still blocking the request with a **“bad origin”** error.

---

## 🕵️‍♂️ Debugging Steps

### 1. CSRF Token & Cookie Logging

Custom middleware was added to log incoming requests:

```go
loggingWrapper := func(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {

        log.Printf("[DEBUG] Method: %s, Path: %s", r.Method, r.URL.Path)

        // Log CSRF Cookie
        cookie, err := r.Cookie("_gorilla_csrf")
        if err != nil {
            log.Println("[DEBUG] CSRF cookie not found:", err)
        } else {
            log.Println("[DEBUG] CSRF cookie:", cookie.Value)
        }

        // Log Form Body or Header Token
        r.ParseForm()
        log.Println("[DEBUG] CSRF form token:", r.FormValue("gorilla.csrf.Token"))
        log.Printf("[DEBUG] Origin header: %s", r.Header.Get("Origin"))
        log.Printf("[DEBUG] Referer header: %s", r.Header.Get("Referer"))
        log.Printf("[DEBUG] Host: %s", r.Host) //To get the Host header from an incoming HTTP request, use r.Host. A call to r.Header.Get("Host") will always return an empty string for an incoming request.

        // Proceed to next middleware (which is CSRF)
        next.ServeHTTP(w, r)
    })
}
```

### 2. Observed Logs

```
[DEBUG] Method: POST, Path: /signup/users
[DEBUG] CSRF cookie: MTc0NzcyMzM...
[DEBUG] CSRF form token: Fkqiel9L53L...
[DEBUG] Origin header: http://localhost:3000
[DEBUG] Referer header: http://localhost:3000/signup
[DEBUG] Host: localhost:3000
```

### 3. Key Insight: Mismatched Origin and Host

- Origin: `http://localhost:3000`
- Host (backend server): `localhost:3000`

`gorilla/csrf`: When your CSRF middleware checks the Origin header, it looks at the host part of that Origin URL and compares it against your trusted origins list.
    Origin: http://localhost:3000
    Origin host: localhost:3000 so this should be in the trusted origin list

---

## ✅ Final Solution

To explicitly allow requests from the frontend (`localhost:3000`), update the CSRF middleware like this:

```go
csrfKey := "d5e8c78e1345dd1f0bc9c33449bf169c"//Ignore this, use your own
csrfMw := csrf.Protect(
    []byte(csrfKey),
    csrf.Secure(false), 
    csrf.TrustedOrigins([]string{
        "http://localhost:3000",
        "localhost:3000", // -> THe "FIX"
    }),
)

http.ListenAndServe(":3000", loggingWrapper(csrfMw(r)))
```

> ✅ The request is now accepted since the origin is explicitly trusted.

---

## ✅ Checklist for CSRF with Gorilla in Dev

- [x] Generate a CSRF token and send it to the client.
- [x] Include the token in the form or header and cookie in requests.
- [x] Log request headers for debugging.
- [x] Use `csrf.TrustedOrigins([]string{...})` to allow mismatched origins.
- [x] Set `csrf.Secure(false)` when not using HTTPS (dev mode).
