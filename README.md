

# Go Process Engine

[](https://www.google.com/search?q=https://goreportcard.com/report/github.com/your-repo/process)
[](https://www.google.com/search?q=https://godoc.org/github.com/your-repo/process)
[](https://www.google.com/search?q=LICENSE)

A lightweight and flexible Go library for building and running modular, graph-based workflows and data processing pipelines. It's designed to be simple, composable, and easy to extend.

-----

## Core Concepts 💡

The engine is built on a few simple ideas:

  * **Worker**: The fundamental unit of work. A `Worker` is any component that can be executed as part of a larger process. It has a defined lifecycle: `Setup` -\> `Perform` -\> `Teardown`.
  * **Pipeline**: An orchestrator that manages the execution of a graph of `Worker`s. It directs the flow of control from one worker to the next based on their outcomes.
  * **SharedContext**: A simple `map[string]interface{}` that holds data accessible to every worker in a pipeline. It's used for passing state through the entire process.
  * **Chaining**: Workers are linked together using the `.Then()` method to form a directed graph. The result of a worker's `Perform` method (a string) determines which path in the graph is taken next.

-----

## Features ✨

  * **Declarative Pipelines**: Define complex workflows by chaining simple workers together.
  * **Built-in Retry Logic**: `ResilientWorker` automatically retries tasks that fail.
  * **Batch Processing**: `SequentialBatchWorker` and `ConcurrentBatchWorker` for processing slices of data.
  * **Concurrent Execution**: Run multiple pipelines in parallel for high-throughput workloads using `BatchPipeline`.
  * **Customizable**: Easily create your own workers with custom logic by implementing the `Worker` interface.
  * **Type-Safe**: Leverages Go's static typing and explicit error handling.

-----

## Installation

```sh
go get github.com/your-repo/process
```

-----

## Quick Start

Here's how to create and run a simple pipeline that greets a user and checks their status.

### 1\. Define Your Workers

First, create structs for your custom tasks. They should embed one of the base worker types like `process.ResilientWorker`.

```go
package main

import (
	"fmt"
	"github.com/your-repo/process"
	"time"
)

// GreeterWorker says hello.
type GreeterWorker struct {
	process.ResilientWorker
}

func (w *GreeterWorker) Perform(setupResult interface{}) (interface{}, error) {
	config := w.GetConfig()
	name, ok := config["name"].(string)
	if !ok {
		return "error", fmt.Errorf("name not found in config")
	}
	fmt.Printf("Hello, %s!\n", name)

	// This outcome will be used to determine the next worker.
	return "user_validated", nil
}

// StatusCheckerWorker checks the user's status.
type StatusCheckerWorker struct {
	process.ResilientWorker
}

func (w *StatusCheckerWorker) Perform(setupResult interface{}) (interface{}, error) {
	fmt.Println("User status is: Active.")
	return "default", nil // "default" is the standard successful outcome.
}

// ErrorHandlerWorker runs if something goes wrong.
type ErrorHandlerWorker struct {
	process.CoreWorker
}

func (w *ErrorHandlerWorker) Perform(setupResult interface{}) (interface{}, error) {
	fmt.Println("An error occurred during the process.")
	return "default", nil
}
```

### 2\. Build and Run the Pipeline

Now, create instances of your workers and chain them together into a `Pipeline`.

```go
func main() {
	// 1. Create instances of our workers.
	greeter := &GreeterWorker{}
	greeter.Initialize(greeter) // Important: Initialize for virtual methods.

	statusChecker := &StatusCheckerWorker{}
	statusChecker.Initialize(statusChecker)

	errorHandler := &ErrorHandlerWorker{}
	errorHandler.Initialize(errorHandler)

	// 2. Chain the workers together to define the flow.
	// If greeter returns "user_validated", go to statusChecker.
	// If greeter returns "error", go to errorHandler.
	greeter.Then(statusChecker, "user_validated")
	greeter.Then(errorHandler, "error")

	// 3. Create the pipeline with an entry point.
	pipeline := process.NewPipeline(greeter)

	// 4. Set the configuration for this run.
	pipeline.SetConfig(map[string]interface{}{
		"name": "Alice",
	})

	// 5. Start the pipeline!
	fmt.Println("--- Starting Pipeline ---")
	finalOutcome, err := pipeline.Start(nil) // Start with an empty context.
	if err != nil {
		fmt.Printf("Pipeline failed: %v\n", err)
		return
	}

	fmt.Printf("--- Pipeline Finished with outcome: %v ---\n", finalOutcome)
}

// Expected Output:
// --- Starting Pipeline ---
// Hello, Alice!
// User status is: Active.
// --- Pipeline Finished with outcome: default ---
```

-----

## API Overview

### `Worker` Interface

The core interface that defines all runnable components.

```go
type Worker interface {
    Setup(ctx SharedContext) (interface{}, error)
    Perform(setupResult interface{}) (interface{}, error)
    Teardown(ctx SharedContext, setupResult, performResult interface{}) (interface{}, error)
    // ... and more.
}
```

### Base Components

  * `CoreWorker`: The simplest implementation of a `Worker`. Use this for basic, non-retrying tasks.
  * `ResilientWorker`: Embed this to give your worker automatic retry capabilities. Configure it with `MaxAttempts` and `RetryDelay`.
  * `SequentialBatchWorker`: A worker that iterates through a slice and performs its retry-enabled logic on each item, one by one.
  * `ConcurrentBatchWorker`: Like `SequentialBatchWorker`, but processes all items in the slice in parallel using goroutines.

### Orchestrators

  * `Pipeline`: The main orchestrator. You define a graph by chaining workers and set an `EntryPoint` to begin execution.
  * `BatchPipeline`: A powerful component that runs an entire pipeline multiple times based on a list of configurations generated in its `Setup` phase. It can run the pipeline for each configuration either sequentially or concurrently (`Concurrent: true`).
