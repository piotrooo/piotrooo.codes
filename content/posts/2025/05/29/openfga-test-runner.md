---
title: 'OpenFGA Test Runner'
date: 2025-05-29
draft: false
url: 'openfga-test-runner'
description: Test your OpenFGA model.
tldr: |
  Nowadays, you test everything: your code with unit tests, your business logic with integration tests, and your UI with
  rendering engines. Literally everything. So why wouldn’t you test your authorization logic too?
tags: [ ]
---

## What is a OpenFGA?

[OpenFGA](https://openfga.dev/) is an open-source fine-grained authorization system inspired by Google's Zanzibar paper.
It enables developers to define and enforce complex access control policies using a flexible, relationship-based model.
OpenFGA is designed for scalability and performance, supporting millions of users and resources with low latency.

## Why testing your OpenFGA model matters?

Testing your OpenFGA authorization model is crucial to ensure that access control behaves exactly as intended. Since
OpenFGA often manages complex relationships between users, roles, and resources, even a small mistake in the model can
lead to serious security issues — like unauthorized access or denial of service to valid users.

To test models in OpenFGA, you can define authorization scenarios using the built-in authorization model tests format,
which includes tuples, assertions, and expected outcomes. These tests can be run using the FGA CLI. This approach
ensures that all permission rules are enforced correctly before deploying to production.

☝️For more details on how to write and run tests for your OpenFGA models, visit
the [official documentation](https://openfga.dev/docs/modeling/testing).

## Let's automate!

Automated tests help you catch errors early, especially as your authorization policies grow or change over time.
They also serve as living documentation for your model, making it easier to onboard new developers and confidently
refactor policies without fear of introducing regressions. In short, writing tests for OpenFGA is essential to
maintaining secure, predictable, and auditable access control in your systems.

Running OpenFGA tests using [GitHub Actions](https://github.com/features/actions) is highly effective because it lets
you treat your authorization model like regular code. You can automate testing, validate changes with every commit or
pull request, and catch errors early — ensuring your access control logic stays reliable and secure throughout
development.

## Testing tool

As of now, OpenFGA doesn't include a dedicated testing tool or test runner out of the box. While it supports a test
format for models, developers are left to implement their own way to execute these tests. To bridge that gap, I created
a simple Bash script that runs the test automatically. This lightweight approach makes it easy to integrate tests into
GitHub Actions and ensures your access logic is continuously validated.

{{< callout >}}
The script I created requires the official [FGA CLI](https://openfga.dev/docs/getting-started/cli) tool to be installed.
{{< /callout >}}

Below is a screenshot showing how the test script looks when executed. It clearly displays the status of each test file,
indicating whether it passed or failed, along with useful details. This kind of visual feedback makes it easy to quickly
spot issues in your authorization model and understand what needs to be fixed.

{{< figure src="/images/2025/05/29/1-sample-output.png" title="Figure 1. Sample output from the test runner " >}}

All the examples used in this blog post come from https://github.com/piotrooo/sample-stores, which is a fork of the
[official OpenFGA sample repository](https://github.com/openfga/sample-stores). The original version was kindly provided
by the OpenFGA team, and I've adapted it to include custom tests and scripts for demonstration purposes.

## 🎁 Bonus: GitHub Action

To make the testing process even more usable and automation-friendly, I prepared a
ready-to-use [GitHub Action](https://github.com/marketplace/actions/openfga-test-runner). You can easily add it to your
workflow to automatically run OpenFGA tests on every push or pull request.

Enrich your GitHub Workflow by adding the following action:

```yaml
name: CI

on: [ push ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: piotrooo/openfga-test-runner@v0.0.1
```

You can see it in action for yourself at the following links:

👉 [Run 1 – Root Directory Reference](https://github.com/piotrooo/sample-stores/actions/runs/15351509149/job/43200628048)

In this run, the test runner looks for model files starting from the root directory of the repository.

👉 [Run 2 – Specific Directory Reference](https://github.com/piotrooo/sample-stores/actions/runs/15351538219/job/43200730298)

Here, the runner was configured to look only inside a specific subdirectory, allowing for more targeted testing.

---

🙏 I hope this helps make working with OpenFGA more enjoyable and reliable ❤️.

Happy testing and have fun securing your applications! 🚀
