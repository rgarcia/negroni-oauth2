# negroni-oauth2 [![wercker status](https://app.wercker.com/status/cfc6a7d08ba203b6d40aa0b3bd69b477/s/ "wercker status")](https://app.wercker.com/project/bykey/cfc6a7d08ba203b6d40aa0b3bd69b477)

Allows your Negroni application to support user login via an OAuth 2.0 backend. Requires [`negroni-sessions`](https://github.com/goincremental/negroni-sessions) middleware.

Google, Facebook, LinkedIn and Github sign-in are currently supported.

Once endpoints are provided, this middleware can work with any OAuth 2.0 backend.

## Usage

~~~ go
package main

import (
	"fmt"
	"net/http"

	"github.com/codegangsta/negroni"
	"github.com/goincremental/negroni-oauth2"
	"github.com/goincremental/negroni-sessions"
)

func main() {

	secureMux := http.NewServeMux()

	// Routes that require a logged in user
	// can be protected by using a separate route handler
	// If the user is not authenticated, they will be
	// redirected to the login path.
	secureMux.HandleFunc("/restrict", func(w http.ResponseWriter, req *http.Request) {
		token := oauth2.GetToken(req)
		fmt.Fprintf(w, "OK: %s", token.Access())
	})

	secure := negroni.New()
	secure.Use(oauth2.LoginRequired())
	secure.UseHandler(secureMux)

	n := negroni.New()
	n.Use(sessions.Sessions("my_session", sessions.NewCookieStore([]byte("secret123"))))
	n.Use(oauth2.Google(&oauth2.Options{
		ClientID:     "client_id",
		ClientSecret: "client_secret",
		RedirectURL:  "redirect_url",
		Scopes:       []string{"email"},
	}))

	router := http.NewServeMux()

	//routes added to mux do not require authentication
	router.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		token := oauth2.GetToken(req)
		if token == nil || token.IsExpired() {
			fmt.Fprintf(w, "not logged in, or the access token is expired")
			return
		}
		fmt.Fprintf(w, "logged in")
		return
	})

	//There is probably a nicer way to handle this than repeat the restricted routes again
	//of course, you could use something like gorilla/mux and define prefix / regex etc.
	router.Handle("/restrict", secure)

	n.UseHandler(router)

	n.Run(":3000")
}
~~~

## Auth flow

* `/login` will redirect user to the OAuth 2.0 provider's permissions dialog. If there is a `next` query param provided, user is redirected to the next page afterwards.
* If user agrees to connect, OAuth 2.0 provider will redirect to `/oauth2callback` to let your app to make the handshake. You need to register `/oauth2callback` as a Redirect URL in your application settings.
* `/logout` will log the user out. If there is a `next` query param provided, user is redirected to the next page afterwards.

You can customize the login, logout, oauth2callback and error paths:

~~~ go
oauth2.PathLogin = "/oauth2login"
oauth2.PathLogout = "/oauth2logout"
...
~~~

## Authors
* [David Bochenski](http://github.com/bochenski)

## Derived from [martini-contrib/oauth2](http://github.com/martini-contrib/oauth2)

* [Burcu Dogan](http://github.com/rakyll)
