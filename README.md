[![Build Status](https://app.bitrise.io/app/e6a7166eda823c72/status.svg?token=LACL0_krbTkiMlmi4kBLNA&branch=master)](https://app.bitrise.io/app/e6a7166eda823c72)

# Bitrise OAuth library for Go
This package is a very thin layer over Go's standard [OAuth2 library](https://github.com/golang/oauth2) and the official [Auth0 library](https://github.com/auth0-community/auth0-go) for Go, extending their functionality via introducing an additional layer that handles the initialization and communication with our current authorization provider [Keycloak](https://github.com/keycloak/keycloak).

This package provides both *client-side* and *server-side* wrappers, covering all of our current use-cases. In this document, you may find useful information about the APIs, the custom configuration options, and the usage as well.

## Client
The *client-side* validation logic is located in the `client` package. The package offers a convenient way to gain an access token on the *client-side*. It was achieved by extending Go's standard `http.Client`. It basically holds the necessary parameters for a successful token request (like **client ID**, **client secret**, and the **token URL** of the authorization server). You can use the `AuthProvider` in several different ways to gain an access token. You may find information about each use-case in the API paragraph. One important thing to note is that in this context *client* means service client, participating in service-to-service authentication.

### API
#### `AuthProvider` interface
Describes the possible operations and use-cases of our package.

#### `WithSecret` impl
Implements the `AuthProvider` interface. This class is used to gain an authenticated `http.Client` to make further authenticated *HTTP* calls, or alternatively, a token source can be created as well, but in this case, only the access token can be gained, not a complete authenticated `http.Client`. You can use `HTTPClientOption`s to configure.

##### Fields
- `clientID string` holds the client ID.

- `clientSecret string` holds the client secret.

- `tokenURL string` holds the URL of the authentication service that provides an access token.

- `credentials clientcredentials.Config` holds the parameters above in an `oauth.clientcredentials.Config` instance, used by the underlying *OAuth* library.

##### Methods
- `NewWithSecret(clientID, clientSecret string, scopeOption ScopeOption, opts ...Option) AuthProvider` returns a new instance of `AuthProvider`. Setting the authentication scope is mandatory. Furthermore, it might receive `Option`s as a parameter.

- `TokenSource() oauth2.TokenSource` returns an `oauth.clientcredentials.TokenSource` that returns the token until it expires, automatically refreshing it as necessary using the provided context and the client ID and client secret.

- `UMATokenSource() UMATokenSource` returns an `UMATokenSource` that returns a new token upon everytime a new token is acquired. You might want to use it for authorization purposes.

- `HTTPClient(opts ...HTTPClientOption) *http.Client` returns a preconfigured `http.Client`.

- `ManagedHTTPClient(opts ...HTTPClientOption) *http.Client` returns a preconfigured `http.Client`. Uses a thread-safe map to store the created clients, using the `clientID` + `clientSecret` + `tokenURL` combination as a key. When the function is called, it will try to retrieve an existing instance from the map by the credentials. If found, the instance will be returned. Otherwise, a new instance will be created, saved in the map, and returned.

#### `UMATokenSource`
This is responsible for providing authorization purposes. Similarly to the regular `ouath2.TokenSource`, it has a `Token()` method that returns a new token upon each invocation.

##### Methods
- `Token(claim interface{}, permisson []Permission, audienceConfig config.AudienceConfig) (*oauth2.Token, error)` returns a new token upon each invocation.

#### `Permission`
It represents an authorization permission.


### Options
The package offers wide configurability using Options. You can easily override any parameter by passing the desired Option(s) as constructor arguments. Not only the `AuthProvider` itself has Options, but each use-case has their own Options as well, offering further configuration possibilities.

#### Option
- `WithBaseURL(baseURL string) Option` overrides the base URL of the authentication service.

- `WithRealm(realm string) VOption` overrides the realm.

- `WithScope(aud string, auds ...string) ScopeOption` sets the authentication scope.

#### HTTPClientOption
- `WithContext(ctx context.Context) HTTPClientOption` overrides the HTTP context of the client.

- `WithBaseClient(bc *http.Client) HTTPClientOption` can extend an already existing HTTP client.

### Usage
```go
authProvider := client.NewWithSecret("my-client-id", "my-client-secret")
resp, err := authProvider.ManagedHTTPClient().Get("https://myservice.services.bitrise.io/")
```

## Server
The server-side validation logic is located in the `service` package. You can use the `Validator` in several different ways to validate any request. The supported use-cases are the following:
- **Handler Function** with:
	- Go's default HTTP multiplexer
	- Gorilla's HTTP router called [**gorilla/mux**](https://github.com/gorilla/mux)
- **Middleware** with:
	- Go's default HTTP multiplexer
	- Gorilla's HTTP router called [**gorilla/mux**](https://github.com/gorilla/mux)
- **Middleware Function** and **Handler Function** with Labstack's router called [**echo**](https://github.com/labstack/echo)


### API
#### `ValidatorIntf`
Describes the possible operations and use-cases of our package.

#### `Validator`
Implements the `ValidatorIntf` interface. As its name reflects, this class is responsible for the validation of a request, using an `auth0.Validator` instance.
You can use `ValidatorOption`s to configure.

##### Fields
- `validator JWTValidator` holds the `auth0.JWTValidator` instance, used to validate a request. You may find further information about the `JWTValidator` interface in the next paragraph.

- `baseURL string` holds the base URL of the authentication service.

- `realm string` holds the realm.

- `keyCacher auth0.KeyCacher` holds the *JWK* cacher. By default it can hold **5 keys** at max, for no longer than **2 hours**.

- `jwksURL string` holds the keystore URL.

- `realmURL string` holds the realm URL.

- `signatureAlgorithm jose.SignatureAlgorithm` holds the encryption/decription algorithm of the *JWT*. By default this is `RS256`.

- `timeout time.Duration` holds the timeout duration. By default this is **30 seconds**.

##### Methods
- `NewValidator(audienceConfig AudienceConfig, opts ...ValidatorOption) ValidatorIntf` returns a new instance of `Validator`. Setting the expected audience is mandatory. Furthermore, it might receive `ValidatorOption`s as a parameter.

- `ValidateRequest(r *http.Request) error` calls the `ValidateRequest` function of `auth0.JWTValidator` instance to validate a request. It returns `nil` if the validation has succeeded, otherwise returns an `error`.

- `ValidateRequestAndReturnToken(r *http.Request) (*TokenWithClaims, error)` calls the `ValidateRequest` function of `auth0.JWTValidator` instance to validate a request. It returns the validated `TokenWithClaims` token, that can be used to get the claims if the validation has succeeded, otherwise returns an `error`.

- `Middleware(next http.Handler, opts ...HTTPMiddlewareOption) http.Handler` returns an `http.Handler` instance. It calls `ValidateRequest` to validate the request. Calls the next middleware if the validation has succeeded, otherwise sends an error using an error writer. It might receive `HTTPMiddlewareOption`s as a parameter.

- `MiddlewareFunc(opts ...EchoMiddlewareOption) echo.MiddlewareFunc` returns an `echo.MiddlewareFunc` instance. It calls `ValidateRequest` to validate the request. Calls the next `echo.HandlerFunc` if the validation has succeeded, otherwise returns an `error`. It might receive `EchoMiddlewareOption`s as a parameter. 

- `HandlerFunc(hf http.HandlerFunc, opts ...HTTPMiddlewareOption) http.HandlerFunc` returns a `http.HandlerFunc` instance. It calls `ValidateRequest` to validate the request. Calls the next handler function if the validation has succeeded, otherwise sends an error using an error writer. It might receive `HTTPMiddlewareOption`s as a parameter.

#### `JWTValidator`
Since `auth0.JWTValidator` is not an interface, it was necessary to create an interface to loosen the coupling and making it exchangeable and mockable in tests.

#### `AudienceConfig`
You can set the expceted audience via passing an `AudienceConfig` instance as a parameter upon instantiating the validator. One or more audience have to passed.

##### Fields
- `audience []string` holds the audiences.

##### Methods
- `NewAudienceConfig(audience string, audiences ...string) AudienceConfig` returns a new `AudienceConfig`. One or more audiences have to be set.

- `All() []string` returns all of the audiences.

##### Usage
```go
validator := service.NewValidator(config.NewAudienceConfig("audience1", "audience2"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))
```

#### `TokenWithClaims`
Represents an UMA token that holds certain claims.

##### Methods
- `Payload() (map[string]interface{}, error)` returns the contents of the token (basically all the claims in the token)

- `Permissions() ([]interface{}, error)` returns the persmissions part of the token.

- `Claim(resourceName string, claim interface{}) error` returns the claim for the provided resource's name.

- `ValidateScopes(scopes []string) error` check if the token has ALL the passed scopes in its scope claim

```go
err := tokenWithClaims.ValidateScopes([]string{"app:read", "missing:write"})
if err != nil {
	// scope validation failed
}
```

### Options
The package offers wide configurability using Options. You can easily override any parameter by passing the desired Option(s) as constructor arguments. Not only the `Validator` itself has Options, but each use-case has their own Options as well, offering further configuration possibilities.

#### ValidatorOption
If you want to override the default options (like the auth service url, or the realm), you can customize the `Validator` during instantiation with Options. It's as easy as passing them as constructor parameters, separated by a comma:
```go
service.NewValidator(config.NewAudienceConfig("audience"),
  service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))
```

The available `ValidatorOption`s are the following:
- `WithBaseURL(url string) ValidatorOption` overrides the base URL of the authentication service.

- `WithSignatureAlgorithm(sa jose.SignatureAlgorithm) ValidatorOption` overrides the encryption/decryption algorithm of the *JWT*.

- `WithRealm(realm string) ValidatorOption` overrides the realm.

- `WithKeyCacher(kc auth0.KeyCacher) ValidatorOption` overrides the *JWK* cacher.

- `WithTimeout(timeout time.Duration) ValidatorOption` overrides the timeout for validation networking.

#### HTTPMiddlewareOption
You can configure the *Handler Function* and *Middleware* use-cases via passing these Options either to `Validator`'s `HandlerFunc` or `Middleware` function. The available `HTTPMiddlewareOption`s are the following:
- `WithHTTPErrorWriter(errorWriter func(w http.ResponseWriter, r *http.Request, err error)) HTTPMiddlewareOption` overrides the error writer.

#### EchoMiddlewareOption
You can configure the *echo* use-case via passing these Options to `Validator`'s `MiddlewareFunc` function. The available `EchoMiddlewareOption`s are the following:
- `WithContextErrorWriter(errorWriter func(echo.Context, error) error) EchoMiddlewareOption` overrides the error writer.


### Usage

#### Validating request/token and extracting claims
```go
func someHandler(w http.ResponseWriter, r *http.Request) {	
	token, err := validator.ValidateRequestAndReturnToken(r)
	if err != nil {
		panic(err)
	}

	claims, err := token.Payload()
	if err != nil {
		panic(err)
	}
	claimsResponse := fmt.Sprintf("%v", claims)

	w.Write([]byte(claimsResponse))
}
```

#### Handler Function
```go
handler := func(w http.ResponseWriter, r *http.Request) {}

mux := http.NewServeMux()

validator := service.NewValidator(config.NewAudienceConfig("audience"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))

mux.HandleFunc("/test_func", validator.HandlerFunc(handler))

log.Fatal(http.ListenAndServe(":8080", mux))
```

#### Handler Function with gorilla/mux
```go
handler := func(w http.ResponseWriter, r *http.Request) {}

router := mux.NewRouter()

validator := service.NewValidator(config.NewAudienceConfig("audience"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))

router.HandleFunc("/test_func", validator.HandlerFunc(handler)).Methods(http.MethodGet)

http.Handle("/", router)

log.Fatal(http.ListenAndServe(":8080", router))
```

#### Middleware
```go
handler := func(w http.ResponseWriter, r *http.Request) {}

mux := http.NewServeMux()

validator := service.NewValidator(config.NewAudienceConfig("audience"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))

mux.Handle("/test", validator.Middleware(http.HandlerFunc(handler)))

log.Fatal(http.ListenAndServe(":8080", mux))
```

#### Middleware with gorilla/mux
```go
handler := func(w http.ResponseWriter, r *http.Request) {}

router := mux.NewRouter()

validator := service.NewValidator(config.NewAudienceConfig("audience"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))

router.Handle("/test", validator.Middleware(http.HandlerFunc(handler))).Methods(http.MethodGet)

http.Handle("/", router)

log.Fatal(http.ListenAndServe(":8080", router))
```

#### Echo Middleware Function
```go
handler := func(c echo.Context) error {
	return c.String(http.StatusOK, "Hello, World!")
}

e := echo.New()

validator := service.NewValidator(config.NewAudienceConfig("audience"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))

e.Use(validator.MiddlewareFunc())

e.GET("/test", handler)

e.Logger.Fatal(e.Start(":8080"))
```

#### Echo Handler Function
```go
validator := service.NewValidator(config.NewAudienceConfig("audience"),
	service.WithBaseURL("https://auth.services.bitrise.io"), service.WithRealm("master"))

handler := func(c echo.Context) error {
	if err := validator.ValidateRequest(c.Request()); err != nil {
		return err
	}
	return c.String(http.StatusOK, "Hello, World!")
}

e := echo.New()

e.GET("/test", handler)

e.Logger.Fatal(e.Start(":8080"))
```
