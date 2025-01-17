# goyek

> Create build pipelines in Go

[![Go Reference](https://pkg.go.dev/badge/github.com/goyek/goyek.svg)](https://pkg.go.dev/github.com/goyek/goyek)
[![Keep a Changelog](https://img.shields.io/badge/changelog-Keep%20a%20Changelog-%23E05735)](CHANGELOG.md)
[![GitHub Release](https://img.shields.io/github/v/release/goyek/goyek)](https://github.com/goyek/goyek/releases)
[![go.mod](https://img.shields.io/github/go-mod/go-version/goyek/goyek)](go.mod)
[![LICENSE](https://img.shields.io/github/license/goyek/goyek)](LICENSE)

[![Build Status](https://img.shields.io/github/workflow/status/goyek/goyek/build)](https://github.com/goyek/goyek/actions?query=workflow%3Abuild+branch%3Amain)
[![Go Report Card](https://goreportcard.com/badge/github.com/goyek/goyek)](https://goreportcard.com/report/github.com/goyek/goyek)
[![codecov](https://codecov.io/gh/goyek/goyek/branch/main/graph/badge.svg)](https://codecov.io/gh/goyek/goyek)
[![Mentioned in Awesome Go](https://awesome.re/mentioned-badge.svg)](https://github.com/avelino/awesome-go)

[![Slack](https://img.shields.io/badge/slack-@gophers/goyek-brightgreen.svg?logo=slack)](https://gophers.slack.com/archives/C020UNUK7LL)

Please ⭐ `Star` this repository if you find it valuable and worth maintaining.

Table of Contents:

- [goyek](#goyek)
  - [Description](#description)
  - [Quick start](#quick-start)
  - [Examples](#examples)
  - [Wrapper scripts](#wrapper-scripts)
  - [Features](#features)
    - [Task registration](#task-registration)
    - [Task command](#task-command)
    - [Task dependencies](#task-dependencies)
    - [Helpers for running programs](#helpers-for-running-programs)
    - [Verbose mode](#verbose-mode)
    - [Default task](#default-task)
    - [Parameters](#parameters)
    - [Supported Go versions](#supported-go-versions)

## Description

**goyek** (/ˈɡɔɪæk/ [🔊 listen](http://ipa-reader.xyz/?text=%CB%88%C9%A1%C9%94%C9%AA%C3%A6k))
is used to create build pipelines in Go.
As opposed to many other tools, it is just a Go library.

Here are some good parts:

- No binary installation is needed. Simply add it to `go.mod` like any other Go module.
  - You can be sure that everyone uses the same version of **goyek**.
- It has low learning curve, thanks to the minimal API surface, documentation, and examples.
- The task's command look like a unit test.
  It is even possible to use [`testify`](https://github.com/stretchr/testify)
  or [`is`](https://github.com/matryer/is) for asserting.
- It is easy to debug, like a regular Go application.
- Tasks and helpers can be easily tested. See [exec_test.go](exec_test.go).
- One can reuse code like in any Go application. It may be helpful to use packages like:
  - [`github.com/bitfield/script`](https://pkg.go.dev/github.com/bitfield/script)
  - [`github.com/rjeczalik/notify`](https://pkg.go.dev/github.com/rjeczalik/notify)
  - [`github.com/magefile/mage/target`](https://pkg.go.dev/github.com/magefile/mage/target)
  - [`github.com/mattn/go-shellwords`](https://pkg.go.dev/github.com/mattn/go-shellwords)

**goyek** API is mainly inspired by the [`testing`](https://golang.org/pkg/testing),
[`http`](https://golang.org/pkg/http), and [`flag`](https://golang.org/pkg/flag) packages.

See [docs/alternatives.md](docs/alternatives.md) if you want to compare **goyek**
with other popular tools used for creating build pipelines.

See [docs/presentations.md](docs/presentations.md) if you want to watch some presenentations.

See [docs/contributing.md](docs/contributing.md) if you want to help us.

## Quick start

Copy and paste the following code into [`build/build.go`](examples/basic/main.go):

```go
package main

import (
	"github.com/goyek/goyek"
)

func main() {
	flow := &goyek.Taskflow{}

	flow.Register(goyek.Task{
		Name:  "hello",
		Usage: "demonstration",
		Command: func(tf *goyek.TF) {
			tf.Log("Hello world!")
		},
	})

	flow.Main()
}
```

Run:

```shell
go mod tidy
```

Sample usage:

```shell
$ go run ./build -h
Usage: [flag(s) | task(s)]...
Flags:
  -v     Default: false    Verbose: log all tasks as they are run.
  -wd    Default: .        Working directory: set the working directory.
Tasks:
  hello    demonstration
```

```shell
$ go run . hello
ok     0.000s
```

```shell
$ go run ./build all -v
===== TASK  hello
Hello world!
----- PASS: hello (0.00s)
ok      0.001s
```

## Examples

- [examples](examples)
- [build/build.go](build/build.go) - this repository's own build pipeline
- [pellared/fluentassert](https://github.com/pellared/fluentassert) - a library using **goyek** without polluting it's root `go.mod`

## Wrapper scripts

Instead of executing `go run ./build`,
you can use use the wrapper scripts,
which can be invoked from any location.

Simply add them to your repository's root directory:

- [`goyek.sh`](goyek.sh) - make sure to add `+x` permission (`git update-index --chmod=+x goyek.sh`):

```bash
#!/bin/bash
set -euo pipefail

DIR="$( cd "$( dirname "${BASH_SOURCE[0]}" )" >/dev/null && pwd )"
cd "$DIR/build"
go run . -wd=".." $@
```

- [`goyek.ps1`](goyek.ps1):

```powershell
$ErrorActionPreference = "Stop"

Push-Location "$PSScriptRoot\build"
& go run . -wd=".." $args
Pop-Location
exit $global:LASTEXITCODE
```

## Features

### Task registration

The registered tasks are required to have a non-empty name, matching
the regular expression `^[a-zA-Z0-9_][a-zA-Z0-9_-]*$`, available as
[`TaskNamePattern`](https://pkg.go.dev/github.com/goyek/goyek#TaskNamePattern).
This means the following are acceptable:

- letters (`a-z` and `A-Z`)
- digits (`0-9`)
- underscore (`_`)
- hyphens (`-`) - except at the beginning

A task with a given name can be only registered once.

A task without description is not listed in CLI usage.

### Task command

Task command is a function which is executed when a task is executed.
It is not required to to set a command.
Not having a command is very handy when registering "pipelines".

### Task dependencies

During task registration it is possible to add a dependency to an already registered task.
When taskflow is processed, it makes sure that the dependency is executed before the current task is run.
Take note that each task will be executed at most once.

### Helpers for running programs

Use [`func (tf *TF) Cmd(name string, args ...string) *exec.Cmd`](https://pkg.go.dev/github.com/goyek/goyek#TF.Cmd)
to run a program inside a task's command.

You can use it create your own helpers, for example:

```go
import (
	"fmt"
	"os/exec"

	"github.com/goyek/goyek"
	"github.com/mattn/go-shellwords"
)

func Cmd(tf *goyek.TF, cmdLine string) *exec.Cmd {
	args, err := shellwords.Parse(cmdLine)
	if err != nil {
		tf.Fatalf("parse command line: %v", err)
	}
	return tf.Cmd(args[0], args[1:]...)
}

func Exec(cmdLine string) func(tf *goyek.TF) {
	args, err := shellwords.Parse(cmdLine)
	if err != nil {
		panic(fmt.Sprintf("parse command line: %v", err))
	}
	return func(tf *goyek.TF) {
		if err := tf.Cmd(args[0], args[1:]...).Run(); err != nil {
			tf.Fatal(err)
		}
	}
}
```

[Here](https://github.com/goyek/goyek/issues/60) is the explantion why argument splitting is not included out-of-the-box.

### Verbose mode

Enable verbose output using the `-v` CLI flag.
It works similar to `go test -v`. Verbose mode streams all logs to the output.
If it is disabled, only logs from failed task are send to the output.

Use [`func (f *Taskflow) VerboseParam() BoolParam`](https://pkg.go.dev/github.com/goyek/goyek#Taskflow.VerboseParam)
if you need to check if verbose mode was set within a task's command.

### Default task

Default task can be assigned via the [`Taskflow.DefaultTask`](https://pkg.go.dev/github.com/goyek/goyek#Taskflow.DefaultTask) field.

When the default task is set, then it is run if no task is provided via CLI.

### Parameters

The parameters can be set via CLI using the flag syntax.

On the CLI, flags can be set in the following ways:

- `-param simple` - for simple single-word values
- `-param "value with blanks"`
- `-param="value with blanks"`
- `-param` - setting boolean parameters implicitly to `true`

For example, `./goyek.sh test -v -pkg ./...` would run the `test` task
with `v` bool parameter (verbose mode) set to `true`,
and `pkg` string parameter set to `"./..."`.

Parameters must first be registered via [`func (f *Taskflow) RegisterValueParam(newValue func() ParamValue, info ParamInfo) ValueParam`](https://pkg.go.dev/github.com/goyek/goyek#Taskflow.RegisterValueParam), or one of the provided methods like [`RegisterStringParam`](https://pkg.go.dev/github.com/goyek/goyek#Taskflow.RegisterStringParam).

The registered parameters are required to have a non-empty name, matching
the regular expression `^[a-zA-Z0-9][a-zA-Z0-9_-]*$`, available as
[`ParamNamePattern`](https://pkg.go.dev/github.com/goyek/goyek#ParamNamePattern).
This means the following are acceptable:

- letters (`a-z` and `A-Z`)
- digits (`0-9`)
- underscore (`_`) - except at the beginning
- hyphens (`-`) - except at the beginning

After registration, tasks need to specify which parameters they will read.
Do this by assigning the [`RegisteredParam`](https://pkg.go.dev/github.com/goyek/goyek#RegisteredParam) instance from the registration result to the [`Task.Params`](https://pkg.go.dev/github.com/goyek/goyek#Task.Params) field.
If a task tries to retrieve the value from an unregistered parameter, the task will fail.

When registration is done, the task's command can retrieve the parameter value using the `Get(*TF)` method from the registration result instance during the task's `Command` execution.

See [examples/parameters/main.go](examples/parameters/main.go) for a detailed example.

`Taskflow` will fail execution if there are unused parameters.

### Supported Go versions

Minimal supported Go version is 1.11.
