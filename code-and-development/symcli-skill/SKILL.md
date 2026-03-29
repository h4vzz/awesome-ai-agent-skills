---
name: SymCLI
description: Execute SymCLI to solve math equations, optimize tensor graphs, or analyze C# code for vulnerabilities. Turn your coding agent into an AI mathematician.
license: MIT
metadata:
  author: Wowo51
  version: 1.0.0
---

# SymCLI

SymCLI is a high-performance symbolic computation engine designed to act as an exact mathematical "System 2" engine for AI agents. It provides deterministic results for algebraic equations, tensor optimization, and C# code analysis, preventing LLM hallucinations in complex logical and mathematical tasks.

## Workflow

1. **Interpret the mathematical or coding task.** Analyze the user's request to determine if it requires symbolic solving, tensor optimization, or C# code auditing.

2. **Formulate the required input.** For mathematical tasks, write a ProblemScript (`.ps`) file with the target variables, rulepacks, and constraints. For C# analysis, identify the target file or directory and the desired audit type (`csharp-math`).

3. **Execute the SymCLI wrapper.** Call the appropriate wrapper script (`symcli.bat` on Windows or `symcli.sh` on Unix-like systems) with the formulated input and a path for the output results.

4. **Read and interpret the results.** Parse the output file (text or JSON) to extract the exact symbolic answers or audit findings. Relate these back to the user's original query with technical precision.

## Supported Tasks

| Task Type | Description | Example Input |
|-----------|-------------|---------------|
| **Algebraic** | Solving polynomials, linear systems, and differential equations. | `x^2 - 5*x + 6 = 0` |
| **Tensor** | Optimizing expression graphs (Fusion, Factoring, Scale Folding). | `MatMul(A, MatMul(B, C))` |
| **C# Audit** | Scanning code for math hazards (`CSMATH`) and security issues (`CSSEC`). | `src/MathCore/Calculator.cs` |
| **Logic** | Solving mixed logic and math constraints. | `x > 0 and x < 10 and x^2 = 25` |

## Usage

Provide one or more of the following:

- **ProblemScript content** for solving math (e.g., `<Options>Target: x</Options> x^2 = 4`).
- **C# source file** for analysis (e.g., `analyze csharp-math src/Core/`).
- **RulePack preference** to guide the solver (e.g., `Algebraic`, `Tensor`, `Logic`).

## Examples

### Example 1 â€” Solving an Algebraic Equation

Given the task to solve `x^2 + 2x + 1 = 0`, the agent writes `problem.ps`:

```xml
<Options>
  Target: x
  RulePacks: Algebraic
</Options>
x^2 + 2*x + 1 = 0
```

Run: `symcli.bat problem.ps result.txt`

Output in `result.txt`:
```text
x = -1
```

### Example 2 â€” Analyzing C# Code for Math Hazards

The agent executes a scan for numerical hazards in a specific file:

```bash
symcli.bat analyze csharp-math src/MathCore/Calculator.cs report.json --json
```

The agent then reviews `report.json` for findings like `CSMATH001` (Unchecked floating point comparison) or `CSSEC005` (Insecure random number generator).

## Best Practices

- **Use explicit Target variables.** In ProblemScript, always specify the variables you want to solve for in the `<Options>` block.
- **Select relevant RulePacks.** Use `Algebraic` for standard math, `Tensor` for ML/Graph tasks, and `Logic` for boolean constraints to optimize solver performance.
- **Prefer JSON output for automation.** When running analysis, use the `--json` flag to get machine-readable reports that are easier for agents to parse.
- **Handle symbolic results.** If an equation has no closed-form numeric solution, SymCLI will return the simplified symbolic expression. Present this to the user as the exact mathematical truth.

## Edge Cases

- **Division by zero**: Returns symbolic `Infinity` or `Undefined` and flags it as a potential hazard during C# analysis.
- **Complex Roots**: Supported in the `Algebraic` rulepack; the agent should be prepared to handle `i` in results.
- **Non-deterministic LLM behavior**: SymCLI is deterministic. If the LLM's interpretation of a task changes, the solver will still provide the correct answer for the provided input, allowing for easy debugging of agent reasoning.
- **Unsupported OS**: SymCLI requires a Windows or Unix-like environment with the .NET runtime installed. Ensure the environment is compatible before execution.
