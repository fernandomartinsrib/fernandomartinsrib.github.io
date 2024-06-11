---
layout: post
title:  "Loading environment variables in Go"
date:   2024-06-11 17:15:00 -0300
categories: golang
published: true
tags:
    - golang
    - os
    - godotenv
    - env
    - viper
image:
  path: /assets/img/thumbs/env_vars.png
  lqip: /assets/img/thumbs/env_vars.png
  alt: Loading environment variables in Go.
---

## Why should we set environment variables?

There are multiple ways of loading environment variables and environment files in golang. But why should we configure environment variables if it is possible to define such variables directly in the code?

Well, let's suppose you have an application that accesses any Database, most certainly you need to provide the credentials to create a client connection. To do so, you'd need to hard-code the secrets of the database in the code which might lead you to some issues.

#### Security

* All secrets will be available in the repository, which means that any person with privilegies to read the repository would be able to access the db using the same secrets;

#### Code management

* Any change to any env variable would require the code to be changed. Which adds unnecessary and pointless work;

#### Alternative

* The best approach for such types of variables is to create environment files that will be loaded into environment variables and may be used by the application without compromising the security of the app to people whose reading access was granted.

## OS package

The **os** package provides a platform-independent interface to operating system functionalities. It includes functions for working with files, directories, processes, environment variables, and whatnot.

### os.Environ

This function returns a copy of strings representing the environment variables configured in the system, in the form "key=value". The output is a []string.

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	env := os.Environ()
	fmt.Println(env)
}
```

Output

```shell
$ go run main.go

MANPATH=/opt/homebrew/share/man:: INFOPATH=/opt/homebrew/share/info: P9K_TTY=old _P9K_TTY=/dev/ttys002 P9K_SSH=0 _P9K_SSH_TTY=/dev/ttys002 _=/usr/local/go/bin/go ENVIRONMENT=local
```

### os.Setenv and os.Getenv

* Setenv sets the value of the environment variable named by the key. It returns an error, if any.

* Getenv retrieves the value of the environment variable named by the key. It returns the value, which will be empty if the variable is not present.

```go
package main

import (
	"fmt"
	"os"
)

func main() {
	err := os.Setenv("ENVIRONMENT", "production")
	if err != nil {
		panic("failed to update env variable")
	}

	environment := os.Getenv("ENVIRONMENT")
	fmt.Printf("ENVIRONMENT: %s", environment)
}
```

### os.LookupEnv

LookupEnv retrieves the value of the environment variable named by the key. If the variable is present in the environment the value (which may be empty) is returned and the boolean is true. Otherwise, if the variable is not found the returned value will be empty and the boolean will be false.

```go
package main

import (
	"fmt"
	"os"
)

func GetEnvVariable(key string) string {
	envVar, found := os.LookupEnv(key)
	if !found {
		panic("variable is not set")
	}

	return envVar
}

func main() {
	env := GetEnvVariable("ENVIRONMENT")
	fmt.Println(env)
}
```

## GoDotEnv package

It is not always practical to set environment variables on development machines or continuous integration servers where multiple projects are running at the same time. GoDotEnv loads variables from a .env file into ENV when the environment is bootstrapped.

It can be used as a library or as a bin command. For this tutorial, we're going to use GoDotEnv as a library.

### Installation

```zsh
$ go get github.com/joho/godotenv
```

### Usage

Add your application configuration to your **.env** file in the root of your project:

```
ENVIRONMENT=local
SECRET_KEY=YOURSECRETKEY
```

Then in your Go application, you may load the variables

```go
package main

import (
    "log"
    "os"

    "github.com/joho/godotenv"
)

func main() {
  err := godotenv.Load()
  if err != nil {
    log.Fatal("Error loading .env file")
  }

  environment := os.Getenv("ENVIRONMENT")
  secretKey := os.Getenv("SECRET_KEY")

  ...
}
```

last but not least, if you don't want GoDotEnv computing your env you can just get a map back instead.

```go
var myEnv map[string]string
myEnv, err := godotenv.Read()

secretKey := myEnv["SECRET_KEY"]
```

... or from an **io.Reader** instead of a local file.

```go
reader := getRemoteFile()
myEnv, err := godotenv.Parse(reader)
```

... or from a **string**.

```go
content := getRemoteFileContent()
myEnv, err := godotenv.Unmarshal(content)
```

## Viper package

Viper is a complete configuration solution for Go applications. It is designed to work within an application and can handle several types of configuration needs and formations.

### Installation
```shell
go get github.com/spf13/viper
```

### Establishing Defaults

> A good configuration system will support default values. A default value is not required for a key, but it’s useful in the event that a key hasn't been set via config file, environment variable, remote configuration or flag.

```go
viper.SetDefault("ContentDir", "content")
viper.SetDefault("LayoutDir", "layouts")
viper.SetDefault("Taxonomies", map[string]string{"tag": "tags", "category": "categories"})
```

### Reading config files

Viper requires minimal configuration so it knows where to look for config files. Viper supports JSON, TOML, YAML, HCL, INI, env file and Java Properties files. In this tutorial, we are going to load a .env file.

After loading the environment variables present in your .env file, you just need to call **env.Get** to fetch them individually.

```go
viper.SetConfigType("env")
viper.AddConfigPath(path_of_env_file)

err := viper.ReadInConfig()
if err != nil {
  return config, err
}

secretKey := viper.Get("SECRET_KEY")
```

### Watching and re-reading config files

Viper has the ability to live-read your config files while the application is running. There's no need to restart your app to start using the new configuration. We just have to tell Viper to watch any changes. Also, we can define a function that will run each time a change occurs.

```go
// Testing live changes
viper.OnConfigChange(func(e fsnotify.Event) {
  fmt.Println("config file changed: ", e.Name)
})

viper.WatchConfig()
```

## Parsing environment variables

Loading .env variables from env files is pretty cool, isn't it? However, I'm pretty sure that you don't want to just load them, but you most certainly want to have a struct containing all the env variables your application will use in execution time. To do so, let's learn how to parse them properly using the following approaches.

## Using env package

### Installation

Get the module with:

```shell
go get github.com/caarlos0/env/v10
```

### Usage

The usage looks like this:

```go
package main

import (
	"fmt"

	"github.com/caarlos0/env/v10"
	"github.com/joho/godotenv"
)

type Config struct {
	Environment string `env:"ENVIRONMENT"`
	SecretKey   string `env:"SECRET_KEY"`
}

func main() {
	if err := godotenv.Load(); err != nil {
		panic("failed to read .env")
	}

	cfg := Config{}
	if err := env.Parse(&cfg); err != nil {
		panic("error parsing env config")
	}

	fmt.Printf("%+v\n", cfg.SecretKey)
}
```

## Using viper package

Similarly to what we did with env, we are going to use viper to unmarshal the variables into a struct.

The only new concept is that we are going to use **mapstructure**.

> mapstructure is a Go library for decoding generic map values to structures and vice versa, while providing helpful error handling. This library is most useful when decoding values from some data stream where you don't quite know the structure of the underlying data until you read a part of it

```go
package main

import (
	"fmt"
	"time"

	"github.com/spf13/viper"
)

type Config struct {
	Environment string `mapstructure:"ENVIRONMENT"`
	DbDriver    string `mapstructure:"DB_DRIVER"`
	DbSource    string `mapstructure:"DB_SOURCE"`
	SecretKey   string `mapstructure:"SECRET_KEY"`
}

func main() {
	var config Config

	viper.SetConfigName("app")
	viper.SetConfigType("env")
	viper.AddConfigPath(".")

	err := viper.ReadInConfig()
	if err != nil {
		panic("couldn't read .env file")
	}

	if err = viper.Unmarshal(&config); err != nil {
		panic("couldn't parse env variables")
	}

	fmt.Println(config.SecretKey)
}
```

## Wrapping up

Well, as we walked through, this tutorial intended to show different ways to load environment variables from environment files. If you want to go deeper don't forget to check out the official documentation for each library we covered here!

## Repositories

A complete example of how to use GoDotEnv to read .env files can be found on this link: [Loading env variables using GoDotEnv](https://gist.github.com/fernandomartinsrib/39024701ff49e6a166c1c5877379aff3)

A complete example of how to use Viper to read .env files can be found on this link: [Loading env variables using Viper](https://gist.github.com/fernandomartinsrib/ee15bd89684de735730df259a8b904f3)


Buy me a coffee if you want to!
[Buy me a coffee](https://www.buymeacoffee.com/fernandomartinsrib)
