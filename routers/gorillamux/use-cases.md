# Use-cases

How to wire OpenAPI spec document and handlers together?

1. OpenAPI spec is created in code

- kin-openapi provides builder functions
- at the end there is a router (mux.Router) and OpenAPI spec (openapi3.T)

```go
type myRequest struct {
    ID string `json:"id"`
    // <here goes meta information to setup request validation, e.g. regular expression>
    FavoriteColor string `json:"favoriteColor"`
}
type myResponse struct {
    PotatoCount int `json:"potatoCount"`
}
type myQueryParams struct {
    Limit int `json:"limit"`
    // <validation info goes here as well>
    Offset int `json:"offset"`
}


// DefaultOptions are specified in the library distribution, contains for example openapi3.Info and other meta data
server := NewServer(DefaultOptions)
server.AddRoute(Route{
    Path: "/params/{x}/{y}/{z}", // path + path params
    Methods: []string{http.MethodPOST},
    RequestBody: myRequest{}, // OpenAPI spec is created based on reflection
    ResponseBody: myResponse{},
    QueryParams: myQueryParams{},
    HeaderParams: myHeaderParams{}, // same as query params
    PathParams: myPathParams{}, // same as query params, it is also checked that Path template matches this structure fields
    Middlewares: ...
    Handler: http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        var qp myQueryParams
        err := DecodeQueryParams(&qp, r) // query params are decoded to provided structure

        var hp myHeaderParams
        err := DecodeHeaders(&hp, r) // ...

        fmt.Fprintf(w, "hello world from POST handler")
    }),
})

doc := server.OpenAPISpec() // gives open api spec openapi3.T
server.ExposeSwaggerUI("/openapi") // exposes swagger-ui at provided path
err := server.ListenAndServe(":8000")
```

(+) OpenAPI spec is created as a route is defined
(+) Tooling can validate request body against the spec
(+) Request/Response specification is deducted from provided values with code reflection, same struct can be used for unmarshaling in the handler itself
(-) Endpoint inputs and outputs can't be guaranteed runtime in the handler function signature (http.Handler interface is used) therefor tooling doesn't provide compile time assistance
(+) Optional runtime response validation which can be useful for monitoring / error prevention
(-) Limitations: basic path (and method) matching handlers only, no subrouters for middlware grouping
I guess this is maximum what can be done without generics or code generation

2. OpenAPI is defined (in code or in file), handlers are attached later

- useful when there is existing specification for which a server has to be build
- kin-openapi provides a method to attach a handler, display missing handlers (which operations from OpenAPI don't have a handler attached)

```go
// doc can be also unmarshaled from file
doc := openapi3.T{
    OpenAPI: "3.0.0",
    Info: &openapi3.Info{
        Title:   "MyAPI",
        Version: "0.1",
    },
    Paths: openapi3.Paths
        "/params/{x}/{y}/{z}": &openapi3.PathItem{
            Get: &openapi3.Operation{Responses: openapi3.NewResponses()},
            Post: &openapi3.Operation{Responses: openapi3.NewResponses()},
            Parameters: openapi3.Parameters{
                &openapi3.ParameterRef{Value: openapi3.NewPathParameter("x")},
                &openapi3.ParameterRef{Value: openapi3.NewPathParameter("y")},
                &openapi3.ParameterRef{Value: openapi3.NewPathParameter("z")},
            },
        },
    },
}

myGETHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello world from GET handler")
})
myPOSTHandler := http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "hello world from POST handler")
})

server := NewServer(&doc)
server.AttachHandler("/params/{x}/{y}/{z}", http.MethodGET, myGETHandler)
server.AttachHandler("/params/{x}/{y}/{z}", http.MethodPOST, myPOSTHandler)

missingHandlers := server.MissingHandlers() // returns list of missing handlers

server.ExposeSwaggerUI("/openapi") // exposes swagger-ui at provided path
err := server.ListenAndServe(":8000")

```

(+) You can create a server to already known OpenAPI specification
(+) You can see if all operations defined in the spec are covered with handlers
(+) Tooling can validate request body against the spec
(-) No help with parsing request body and/or marshaling response based on the spec
