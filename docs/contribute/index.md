---
myst:
  html_meta:
    description: "Contribute to Livepatch documentation. Learn how to improve docs, give back to the community, and expand your technical skills."
---

(contribute-to-docs)=

# Contribute to our documentation

We warmly welcome your engagement with the Livepatch community and appreciate all contributions, suggestions, and feedback. There are many reasons why you should contribute to the Livepatch documentation.

- **Improve your skills**

    Contributing to the Livepatch documentation is a great way to improve your documentation and technical communication skills. You will get experience writing clear, concise documentation that is helpful to the Livepatch community, and you will have the opportunity to learn about writing documentation that focuses on user needs.

- **Give back to the community**

    By contributing to the Livepatch documentation, you foster a supportive community and can help other users learn about Livepatch. Your contributions make a difference to other Livepatch users.

- **Learn more about Livepatch**

    Contributing to the Livepatch documentation can help you broaden your understanding of Livepatch and its related technologies. Writing documentation often involves exploring new features and investigating potential problems or challenges users may face, which can help you learn more about how Livepatch works and how users interact with it.

We believe that everyone has something valuable to contribute, no matter your level of experience, and we hope to make it as easy as possible to contribute. If you find any part of our process does not work well for you, please let us know.

## Prerequisites

There are some prerequisites to contributing to the Livepatch documentation.

- **Code of Conduct**: You will need to read and agree to the Ubuntu [Code of Conduct](https://ubuntu.com/community/ethos/code-of-conduct). By participating, you implicitly agree to abide by the Code of Conduct.

- **GitHub account**: You need a [GitHub account](https://github.com/) to create issues, comment, reply, or submit contributions.

    You do not need to know git before you start, and you definitely do not need to work on the command line if you do not want to. Many documentation tasks can be done using [GitHub's web interface](https://docs.github.com/en/repositories/working-with-files/managing-files/editing-files). On the command line, we use the standard "fork and pull" process.

- **Licensing**: The first time you contribute to a Canonical project, you will need to sign the Canonical License agreement (CLA). If you have already signed it, for example when contributing to another Canonical project, you do not need to sign it again.

    This license protects your copyright over your contributions, including the right to use them elsewhere, but grants us (Canonical) permission to use them in our project. You can read [more about the CLA](https://ubuntu.com/legal/contributors) before you [sign the CLA](https://ubuntu.com/legal/contributors/agreement).

## Livepatch Docs overview and Diátaxis

The Livepatch documentation is [hosted in GitHub](https://github.com/canonical/livepatch-docs) and rendered on [Read the Docs](https://about.readthedocs.com/). You need to create a GitHub account to contribute, but you do not need a Read the Docs account.

We use the [Sphinx documentation generator](https://www.sphinx-doc.org/) to create our documentation, which is written in [MyST Markdown](https://mystmd.org/) and built from [Canonical's Sphinx Starter Pack](https://github.com/canonical/sphinx-docs-starter-pack). For further details, see the [Starter Pack documentation](https://canonical-starter-pack.readthedocs-hosted.com/stable/).

Our navigational structure, style, and the content of our documentation follows the [Diátaxis](https://diataxis.fr/) systematic framework for technical documentation authoring. This splits documentation pages into tutorials, how-to guides, reference material and explanatory text:

- **Tutorials** are lessons that accomplish specific tasks through *doing*. They help with familiarity and place users in the safe hands of an instructor.
- **How-to guides** are recipes, showing users how to achieve something, helping them get something done. A *How-to guide* has no obligation to teach.
- **Reference** material is descriptive, providing facts about functionality that is isolated from what needs to be done.
- **Explanation** is discussion, helping users gain a deeper or better understanding of Livepatch, including *how* and *why* Livepatch functions the way it does.

For further details on our Diátaxis strategy, see [Diátaxis, a new foundation for Canonical documentation](https://ubuntu.com/blog/diataxis-a-new-foundation-for-canonical-documentation).

Improving our documentation and applying the principles of Diátaxis are on-going tasks. There is a lot to do, and we do not want to deter anyone from contributing to our docs. If you do not know whether something should be a tutorial, how-to guide, reference doc or explanatory text, either ask on the forum or publish what you are thinking. Changes are easy to make, and every contribution helps.

## Ways to contribute

Most Livepatch documentation contributions are made on GitHub. There are several ways you can contribute:

- [Find an issue](https://github.com/canonical/livepatch-docs/issues) or [create your own issue](https://github.com/canonical/livepatch-docs/issues/new) to work on
- Fix an issue directly and [create a pull request (PR)](https://github.com/canonical/livepatch-docs/pulls)
- Report a bug or provide feedback by [creating an issue](https://github.com/canonical/livepatch-docs/issues/new) in GitHub
- Ask a question or help other Livepatch community members on [Discourse](https://discourse.ubuntu.com/)

You can also use the "Give feedback" button at the top of every documentation page to report a problem. Please give us as much information as you can to help us address the problem.

## How to contribute

If you are new to contributing with Git/GitHub or the command line, see the [Getting Started guide](https://canonical-open-documentation-academy.readthedocs.io/en/latest/docs/howto/get-started/using_git/) from the Canonical Open Documentation Academy for an overview.

This section provides basic instructions of how to contribute to our documentation and build locally using GitHub and the command line.

### Prerequisites

To build the documentation locally, you will first need to install some necessary dependencies on your system with the following commands:

```bash
sudo apt update
sudo apt install make python3 python3-venv python3-pip
```

### Build, preview, and test the documentation

We use `make` to build, preview, and test the documentation. You can see all the options in the Makefile, but here is the basic process:

#### Build

To build the docs:

```bash
make html
```

The first time you run this command, it will build everything. This includes installing all of the Python dependencies into the Python virtual environment. When you run it again, it will only update the changed files. This is usually fine, but sometimes causes issues if you have made a lot of structural changes to the documentation. You can also run a clean build, which deletes the existing output files and the Python environment, and then builds the full documentation again.

To run a clean build:

```bash
make clean html
```

#### Preview

After your build succeeds, preview it locally to see how the published documentation will look:

```bash
make run
```

This command builds the documentation and serves it at `http://127.0.0.1:8000/`. When you change a documentation file and save it, the documentation will be automatically rebuilt and refreshed in the browser.

Previewing the docs makes it easier to see how the finalized, published documentation will look. For most changes, you should preview your docs so you can make sure everything (especially the formatting) looks as you expected it to.

#### Test

You can do this before or after previewing the docs. We have two main tests which need to pass for our documentation:

```bash
make spelling
make linkcheck
```

These tests make sure everything is spelled correctly and that all links go to a valid URL. Note that these tests will also be run automatically on your pull request in GitHub.

### Submit a pull request

Once you have finished creating and testing your changes, create a pull request against the [Livepatch documentation repository](https://github.com/canonical/livepatch-docs).

Your submitted issues and pull requests will be reviewed in due time. If you submit a PR, we have some automatic checks that will run against your PR to check for consistent style and language. However, do not let this be a barrier to your contribution. You can still submit contributions to the best of your ability, and if something is inconsistent, we will help you fix it.

## Contributions we accept

We especially value contributions that improve the documentation's clarity, accuracy, and usefulness.

If the issue is one of the following small issues, and you want to fix it yourself, you can submit a pull request with your changes directly in the [GitHub web interface](https://docs.github.com/en/repositories/working-with-files/managing-files/editing-files):

- Typo corrections
- Minor technical corrections
- Broken link fixes
- Small clarifications

For changes larger than those, we require contributor PRs to be tied to an issue. Examples of changes that require an issue include:

- Adding new sections or pages
- Restructuring existing pages
- Extensive rewording
- Significant technical corrections or updates
- Feature additions or enhancements

## Contributions we do not accept

We reserve the right to reject any contribution at our own discretion where:

The outline of proposed work has not already been agreed by a maintainer.
: To protect maintainers' limited time, we do not accept unsolicited contributions. A PR must be tied to a valid and agreed issue unless it only makes a small change. Submitting an issue after a PR does not guarantee that we will then accept the PR.

The pull request far exceeds the scope of what was agreed.
: "Scope creep" makes it much harder to review and accept changes. If a pull request becomes too large, we may ask you to break it into multiple smaller PRs. Stick to the scope of the original issue as much as you can -- if you discover more changes need to be made, discuss them in the PR or the original issue. A new PR can be submitted with the extra fixes.

The contribution makes changes without value.
: To have value, changes must be a *specific and concrete improvement* over what already exists. We do not accept generic "polishing" (for example, substituting synonyms, rewording without adding clarity). If your contribution contains *some* of these, we will ask you to undo them -- but will gladly accept the remaining parts of your pull request that are real improvements. If your contribution *only* contains low-value changes, it will be rejected.

We deem the contribution to be AI generated.
: We do not want any part of our documentation to look or feel like it was produced by AI. Therefore, if what you submitted cannot be easily distinguished from AI output, it does not meet our contribution requirements.

The issue the pull request is addressing was assigned to someone else.
: We want to make this a safe place for contributors to work on issues in their own time and without pressure. Please treat others as you would want to be treated. If you like an issue someone else is working on, let us know and we will create a similar one for you.

## AI policy

The Livepatch documentation has been created by humans, for humans. The occasional "jagged edges" you might find (typos, slightly unconventional wording, humour) represent the very human voices of the people who collaborated to create it. We do not want to average those away by passing the documentation through LLMs.

This policy exists to protect our readers, our community of contributors, and the project maintainers. As AI continues to evolve, we will update these guidelines accordingly.

(contribute-acceptable-use-of-ai)=

### Acceptable use of AI

We do not ban our contributors from using AI. However, we do expect anyone contributing to this project to use it responsibly, as a helper, and not as something that replaces them. Some examples of acceptable usage include:

- Using AI as a spellchecker or grammar helper
- Using it to help with translation into English
- Using it to help outline and organize a draft
- Using it adversarially, to point out missing sections
- Ensuring formatting adheres to our style guide

We want to be open and transparent with our users about the way in which AI is or is not used in our documentation. We (the maintainers) have been experimenting with the ways in which AI can be used to improve the experience for our readers, and we do not object to contributors doing the same -- with the following caveats:

1. **Agreement**: Any such work you do *must* be tied to an issue, and be agreed as valid and necessary with the maintainers *before* you start. Only maintainers are exempt from this rule, since they are most familiar with the project and its scope.

2. **Honesty**: If you use an "AI assist" when making a contribution, you *must* disclose that you used it, and how you used it in the pull request description.

3. **Accessibility**: Issues marked as "good first issues" and "Open Documentation Academy" are *specifically* created and intended for people new to Open Source to get started in a safe place. These issues are easy to complete, well-described, and self-contained for new contributors to learn from. These issues are not for AI.

4. **Responsibility**: Make sure that even if you use AI, you are still adding value based on your own personal skills and competency. You must understand what you are submitting, even if you used an LLM to help you.

## Thank you

Lastly, we would like to thank you for spending your time to help make the Livepatch documentation better. Every step in the right direction is a step worth taking, no matter how large or small.

```{toctree}
:maxdepth: 1
:hidden:

self
```