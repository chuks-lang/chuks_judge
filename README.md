# chuks_judge

**LLM-as-a-Judge evaluation engine for Chuks.** Parallel evaluation, multi-dimensional scoring, A/B comparison, regression testing, and rich reports — all with native concurrency.

## Installation

```bash
chuks add @chuks/judge
```

## Quick Start

```chuks
import { judge, EvalCase } from "pkg/@chuks/judge"
import { ai } from "pkg/@chuks/ai"

// Build the judge model in your entry module and inject it. The judge package
// itself does not import chuks_ai — this keeps it AOT-clean and lets you swap
// in any object exposing async `structured(prompt, schema)` (e.g. a test mock).
var model = ai.openai("gpt-4o").system("You are a strict evaluator.").temperature(0.0)

var evaluator = judge.create({
    model: model,
    modelName: "openai:gpt-4o",   // optional label recorded on each result
    criteria: "accuracy, completeness, and clarity",
    threshold: 0.7
})

var cases = [
    new EvalCase({
        id: "1",
        input: "What is the capital of France?",
        expected: "Paris",
        actual: "The capital of France is Paris.",
        tags: ["geography", "factual"]
    }),
    new EvalCase({
        id: "2",
        input: "Explain recursion",
        expected: "A function that calls itself with a base case",
        actual: "Recursion is when a function calls itself. It needs a base case to stop.",
        tags: ["cs", "concepts"]
    })
]

var report = await evaluator.runAll(cases)
report.print()
```

## Configuration

```chuks
var evaluator = judge.create({
    // Model — an object exposing async structured(prompt, schema), built with
    // chuks_ai (ai.openai / ai.anthropic / ai.google / ai.mistral / ai.ollama /
    // ai.groq) and injected here. See "Supported Providers" below.
    model: ai.openai("gpt-4o"),
    modelName: "openai:gpt-4o",        // optional label recorded on each result

    // Evaluation criteria
    criteria: "accuracy, completeness, and helpfulness",
    rubric: "Score 1.0 for perfect, 0.5 for partial, 0.0 for wrong",
    threshold: 0.7,                    // Pass/fail cutoff (default 0.7)
    scoreType: "numeric",              // "numeric" (0.0-1.0) or "binary" (pass/fail)

    // Parallelism
    workers: 8,                        // Concurrent workers (default: os.cpus())
    rateLimit: 50,                     // Max requests/sec (default: unlimited)
    batchSize: 10,                     // Cases per batch (default: 10)
    maxRetries: 2,                     // Retries on failure (default: 2)

    // Model settings
    temperature: 0.0,                  // Judge temperature (default: 0.0)
    maxTokens: 1024,                   // Max response tokens (default: 1024)

    // Advanced
    dimensions: [],                    // Multi-dimensional scoring configs
    fewShot: [],                       // Calibration examples
    callbacks: {}                      // Event hooks
})
```

### Supported Providers

| Provider  | Model Examples                           | Config Key                             |
| --------- | ---------------------------------------- | -------------------------------------- |
| OpenAI    | gpt-4o, gpt-4-turbo, o1                  | `"openai:gpt-4o"`                      |
| Anthropic | claude-sonnet-4-20250514, claude-3-haiku | `"anthropic:claude-sonnet-4-20250514"` |
| Google    | gemini-1.5-pro, gemini-2.0-flash         | `"google:gemini-1.5-pro"`              |
| Mistral   | mistral-large, codestral                 | `"mistral:mistral-large"`              |
| Ollama    | llama3, mistral, phi                     | `"ollama:llama3"`                      |
| Groq      | llama3-70b-8192, mixtral-8x7b            | `"groq:llama3-70b-8192"`               |

## API

### Single Evaluation

```chuks
var result = await evaluator.evaluate(new EvalCase({
    input: "What is 2+2?",
    expected: "4",
    actual: "The answer is 4."
}))

println(result.score)       // 0.95
println(result.passed)      // true
println(result.reasoning)   // "The response correctly states..."
println(result.issues)      // []
```

### Batch Evaluation (Parallel)

```chuks
var report = await evaluator.runAll(cases)

// Aggregate metrics
println(report.passRate)     // 85.0
println(report.avgScore)     // 0.82
println(report.medianScore)  // 0.85
println(report.totalTokens)  // 12450
println(report.casesPerSec)  // 15.2
```

### Streaming Results

```chuks
import { Channel } from "std/channel"
import { EvalResult } from "pkg/@chuks/judge"

var ch: Channel<EvalResult> = new Channel<EvalResult>(100)

spawn async function() {
    while (true) {
        var result: EvalResult? = ch.receive()
        if (result == null) { break }
        println("Case " + result.id + ": " + string(result.score))
    }
}()

var report = await evaluator.streamAll(cases, ch)
```

### A/B Comparison

```chuks
import { PairwiseCase } from "pkg/@chuks/judge"

var pairCases = [
    new PairwiseCase("What is ML?", responseFromGPT4, responseFromClaude),
    new PairwiseCase("Explain TCP/IP", responseFromGPT4, responseFromClaude)
]

var comparison = await evaluator.compare(pairCases)
println(comparison.winner)      // "B"
println(comparison.aWinRate)    // 40.0
println(comparison.bWinRate)    // 55.0
println(comparison.improvement) // 0.12
```

### Multi-Dimensional Scoring

```chuks
import { DimensionConfig } from "pkg/@chuks/judge"

var evaluator = judge.create({
    model: ai.openai("gpt-4o"),
    modelName: "openai:gpt-4o",
    criteria: "overall quality",
    threshold: 0.7,
    dimensions: [
        new DimensionConfig({ name: "accuracy", weight: 2.0, threshold: 0.8, criteria: "factual correctness" }),
        new DimensionConfig({ name: "clarity", weight: 1.5, threshold: 0.7, criteria: "clear and readable" }),
        new DimensionConfig({ name: "completeness", weight: 1.0, threshold: 0.6, criteria: "covers all aspects" })
    ]
})

var report = await evaluator.runAll(cases)

// Per-dimension metrics
for (var i = 0; i < report.dimensions.length; i = i + 1) {
    var dim = report.dimensions[i]
    println(dim.name + ": " + string(dim.avgScore) + " (" + string(dim.passRate) + "% pass)")
}
```

### Few-Shot Calibration

```chuks
import { FewShotExample } from "pkg/@chuks/judge"

var evaluator = judge.create({
    model: ai.openai("gpt-4o"),
    modelName: "openai:gpt-4o",
    criteria: "accuracy",
    fewShot: [
        new FewShotExample({
            input: "What is 2+2?",
            expected: "4",
            actual: "4",
            score: 1.0,
            reasoning: "Exact match, fully correct"
        }),
        new FewShotExample({
            input: "What is 2+2?",
            expected: "4",
            actual: "About 4, roughly",
            score: 0.6,
            reasoning: "Correct answer but hedging language reduces confidence"
        })
    ]
})
```

### Regression Testing

```chuks
// Save a baseline after a good run
var report = await evaluator.runAll(cases)
report.saveBaseline("v1.0")

// Later, compare against baseline
var newReport = await evaluator.runAll(updatedCases)
var regression = newReport.compareToBaseline("v1.0")

println(regression.passRateDelta)        // -2.5 (dropped 2.5%)
println(regression.regressions.length)   // 3 cases regressed
println(regression.improvements.length)  // 5 cases improved
println(regression.isRegression)         // false (within tolerance)
```

### Report Analysis

```chuks
// Filter and group
var failedCases = report.filter(function(r) { return !r.passed })
var groups = report.groupBy("category")
var worst = report.bottom(5)
var best = report.top(5)
var geoResults = report.withTag("geography")

// Export
report.print()                           // Console summary
await report.toJSON("results.json")      // JSON file
await report.toHTML("report.html")       // Styled HTML report
var md = report.toMarkdown()             // Markdown string
```

### Callbacks

```chuks
var evaluator = judge.create({
    model: ai.openai("gpt-4o"),
    modelName: "openai:gpt-4o",
    criteria: "accuracy",
    callbacks: {
        onResult: function(result) {
            println("✓ " + result.id + " → " + string(result.score))
        },
        onBatch: function(info) {
            println("Batch " + string(info.batchNumber) + "/" + string(info.totalBatches))
        },
        onError: function(err) {
            println("⚠ Error: " + err.message + " (retry " + string(err.retry) + ")")
        },
        onComplete: function(report) {
            println("Done! Pass rate: " + string(report.passRate) + "%")
        }
    }
})
```

## CI Pipeline Example

```chuks
import { judge, EvalCase } from "pkg/@chuks/judge"
import { ai } from "pkg/@chuks/ai"
import { json } from "std/json"
import { fs } from "std/fs"
import { process } from "std/process"

var evaluator = judge.create({
    model: ai.openai("gpt-4o"),
    modelName: "openai:gpt-4o",
    criteria: "accuracy and completeness",
    threshold: 0.8,
    workers: 4
})

// Load test cases from JSON file
var casesData = json.parse(fs.readFile("eval_cases.json"))
var cases = []
for (var i = 0; i < casesData.length; i = i + 1) {
    cases.push(new EvalCase(casesData[i]))
}

var report = await evaluator.runAll(cases)
report.print()
await report.toJSON("eval_results.json")

// Regression check
var regression = report.compareToBaseline("production")
if (regression.isRegression) {
    println("REGRESSION DETECTED — " + string(regression.regressions.length) + " cases regressed")
    process.exit(1)
}

// Save as new baseline if all good
report.saveBaseline("production")
println("All evaluations passed.")
```

## Types Reference

| Type               | Description                                                          |
| ------------------ | -------------------------------------------------------------------- |
| `EvalCase`         | Input test case with input, actual, expected, tags, category, weight |
| `EvalResult`       | Full evaluation result with score, dimensions, reasoning, issues     |
| `DimScore`         | Per-dimension score (name, score, threshold, passed, reasoning)      |
| `DimensionConfig`  | Dimension configuration (name, weight, threshold, criteria)          |
| `FewShotExample`   | Calibration example (input, expected, actual, score, reasoning)      |
| `PairwiseCase`     | A/B comparison input (input, outputA, outputB)                       |
| `PairwiseResult`   | A/B comparison result (winner, scoreA, scoreB, reasoning)            |
| `EvalReport`       | Aggregated report with metrics, filters, grouping, and export        |
| `ComparisonReport` | A/B comparison summary (winner, win rates, scores)                   |
| `RegressionReport` | Baseline comparison (deltas, regressions, improvements)              |

## License

MIT
