<div align="center">

![logo](logo.svg)

A lightweight package for loading environment variables into structs

[![awesome](https://awesome.re/badge.svg)](https://github.com/avelino/awesome-go#configuration)
[![ci](https://github.com/junk1tm/env/actions/workflows/go.yml/badge.svg)](https://github.com/junk1tm/env/actions/workflows/go.yml)
[![docs](https://pkg.go.dev/badge/github.com/junk1tm/env.svg)](https://pkg.go.dev/github.com/junk1tm/env)
[![report](https://goreportcard.com/badge/github.com/junk1tm/env)](https://goreportcard.com/report/github.com/junk1tm/env)
[![codecov](https://codecov.io/gh/junk1tm/env/branch/main/graph/badge.svg)](https://codecov.io/gh/junk1tm/env)

</div>

## 📌 About

This package is made for apps that [store config in environment variables][1].
Its purpose is to replace multiple fragmented `os.Getenv` calls in `main.go`
with a single struct definition, which simplifies config management and improves
code readability.

## 📦 Install

```shell
go get github.com/junk1tm/env
```

## 🚀 Features

* Simple API
* Dependency-free
* Custom [providers](#provider)
* Global [prefix option](#prefix)
* Per-variable [options](#tag-level-options): `required`, `expand`
* Auto-generated [usage message](#usage-on-error)

## 🔧 Usage

`Load` is the main function of this package. It loads environment variables into
the provided struct.

The struct fields must have the `env:"VAR"` struct tag, where `VAR` is the name
of the corresponding environment variable. Unexported fields and fields without
this tag (except nested structs) are ignored.

```go
os.Setenv("PORT", "8080")

var cfg struct {
    Port int `env:"PORT"`
}
if err := env.Load(&cfg); err != nil {
    // handle error
}

fmt.Println(cfg.Port) // 8080
```

Why not just resolve the name automatically, like `toUpperSnakeCase(fieldName)`?
It feels [too clever][2] to me :)

### Supported types

The following types are supported as struct fields:

* `int` (any kind)
* `float` (any kind)
* `bool`
* `string`
* `time.Duration`
* `encoding.TextUnmarshaler`
* slices of any type above

See the `strconv` package from the standard library for parsing rules.

### Default values

Default values can be specified either using the `default` struct tag (has a
higher priority) or by initializing the struct fields directly.

```go
cfg := struct {
    Host string `env:"HOST" default:"localhost"` // either use the `default` tag...
    Port int    `env:"PORT"`
}{
    Port: 8080, // ...or initialize the struct field directly.
}
if err := env.Load(&cfg); err != nil {
    // handle error
}

fmt.Println(cfg.Host) // localhost
fmt.Println(cfg.Port) // 8080
```

### Nested structs

Nested structs of any depth level are supported, but only non-struct fields are
considered as targets for parsing.

```go
os.Setenv("HTTP_PORT", "8080")

var cfg struct {
    HTTP struct {
        Port int `env:"HTTP_PORT"`
    }
}
if err := env.Load(&cfg); err != nil {
    // handle error
}

fmt.Println(cfg.HTTP.Port) // 8080
```

## ✨ Customization

### Provider

`Load` retrieves environment variables values directly from OS. To use a
different source, try `LoadFrom` that accepts an implementation of the
`Provider` interface as the first argument.

```go
// Provider represents an entity that is able to provide environment variables.
type Provider interface {
    // LookupEnv retrieves the value of the environment variable named by the
    // key. If it is not found, the boolean will be false.
    LookupEnv(key string) (value string, ok bool)
}
```

`Map` is a built-in `Provider` implementation that might be useful in tests.

```go
m := env.Map{"PORT": "8080"}

var cfg struct {
    Port int `env:"PORT"`
}
if err := env.LoadFrom(m, &cfg); err != nil {
    // handle error
}

fmt.Println(cfg.Port) // 8080
```

### Tag-level options

The name of the environment variable can be followed by comma-separated options
in the form of `env:"VAR,option1,option2,..."`. The following tag-level options
are supported:

#### Required

Use the `required` option to mark the environment variable as required. In case
no such variable is found, an error of type `NotSetError` will be returned.

```go
// os.Setenv("HOST", "localhost")
// os.Setenv("PORT", "8080")

var cfg struct {
    Host string `env:"HOST,required"`
    Port int    `env:"PORT,required"`
}
if err := env.Load(&cfg); err != nil {
    var notSetErr *env.NotSetError
    if errors.As(err, &notSetErr) {
        fmt.Println(notSetErr.Names) // [HOST PORT]
    }
}
```

#### Expand

Use the `expand` option to automatically expand the value of the environment
variable using `os.Expand`.

```go
os.Setenv("PORT", "8080")
os.Setenv("ADDR", "localhost:${PORT}")

var cfg struct {
	Addr string `env:"ADDR,expand"`
}
if err := env.Load(&cfg); err != nil {
	// handle error
}

fmt.Println(cfg.Addr) // localhost:8080
```

### Function-level options

In addition to the tag-level options, `Load` also supports the following
function-level options:

#### Prefix

It is a common practice to prefix app's environment variables with some string
(e.g., its name). Such a prefix can be set using the `WithPrefix` option:

```go
os.Setenv("APP_PORT", "8080")

var cfg struct {
    Port int `env:"PORT"`
}
if err := env.Load(&cfg, env.WithPrefix("APP_")); err != nil {
    // handle error
}

fmt.Println(cfg.Port) // 8080
```

#### Slice separator

Space is the default separator when parsing slice values. It can be changed
using the `WithSliceSeparator` option:

```go
os.Setenv("PORTS", "8080;8081;8082")

var cfg struct {
    Ports []int `env:"PORTS"`
}
if err := env.Load(&cfg, env.WithSliceSeparator(";")); err != nil {
    // handle error
}

fmt.Println(cfg.Ports[0]) // 8080
fmt.Println(cfg.Ports[1]) // 8081
fmt.Println(cfg.Ports[2]) // 8082
```

#### Strict mode

For cases where most environment variables are required, strict mode is
available, in which all variables without the `default` tag are treated as
required. To enable this mode, use the `WithStrictMode` option:

```go
// os.Setenv("HOST", "localhost")

var cfg struct {
	Host string `env:"HOST"` // (required)
	Port int    `env:"PORT" default:"8080"`
}
if err := env.Load(&cfg, env.WithStrictMode()); err != nil {
	var notSetErr *env.NotSetError
	if errors.As(err, &notSetErr) {
		fmt.Println(notSetErr.Names) // [HOST]
	}
}
```

#### Usage on error

`env` supports printing an auto-generated usage message the same way the `flag`
package does it. It will be printed if the `WithUsageOnError` option is
provided and an error occurs while loading environment variables:

```go
// os.Setenv("DB_HOST", "localhost")
// os.Setenv("DB_PORT", "5432")

cfg := struct {
    DB struct {
        Host string `env:"DB_HOST,required" desc:"database host"`
        Port int    `env:"DB_PORT,required" desc:"database port"`
    }
    HTTPPort int             `env:"HTTP_PORT" desc:"http server port"`
    Timeouts []time.Duration `env:"TIMEOUTS" desc:"timeout steps"`
}{
    HTTPPort: 8080,
    Timeouts: []time.Duration{1 * time.Second, 2 * time.Second, 3 * time.Second},
}
if err := env.Load(&cfg, env.WithUsageOnError(os.Stdout)); err != nil {
    // handle error
}

// Output:
// Usage:
//   DB_HOST    string           required            database host
//   DB_PORT    int              required            database port
//   HTTP_PORT  int              default 8080        http server port
//   TIMEOUTS   []time.Duration  default [1s 2s 3s]  timeout steps
```

[1]: https://12factor.net/config
[2]: https://dave.cheney.net/2019/07/09/clear-is-better-than-clever
