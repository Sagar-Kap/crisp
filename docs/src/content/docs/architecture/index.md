---
title: Architectural Overview
---

The internal logic that powers Crisp is divided into three distinct parts:

1. The **reader** which is responsible for reading the Git commit message either
   from STDIN (piped in) or from the `$GIT_DIR/COMMIT_EDITMSG` file (see the
   [official Git docs](https://git-scm.com/docs/git-commit/2.2.3#Documentation/git-commit.txt-codeGITDIRCOMMITEDITMSGcode)).

2. The `parser` which is responsible for accepting the inputted commit message
   and then parsing it into a Go struct for further data processing.

3. The `validator` which is responsible for running some validation logic on the
   parsed data.

Under the hood, all three components work in tandem to lint your Git commit
messages. The following diagram provides a better understanding of the
underlying logic.

```mermaid
sequenceDiagram
    autonumber

    actor User

    box rgb(117, 115, 109) Crisp Internals
        participant Reader
        participant Parser
        participant Validator
    end

    Note over User,Reader: The commit message linting should<br/>ideally be done using the<br/>Pre-Commit framework!
    User ->> Reader : Make a Git commit

    activate User

    activate Reader
        alt read from STDIN
            Reader -->> Parser : Read from piped input
        else read from COMMIT_EDITMSG
            Reader -->> Parser : Read from file
        end
        opt pass message as flag
            Reader -->> Parser : Read from CLI flag
        end
    deactivate Reader

    activate Parser
        Parser -->> Parser : Parse the message into a struct
        Parser -->> Validator : Crisp validates the message
    deactivate Parser

    activate Validator

            Validator ->> User : Crisp responds with valid/invalid message
    deactivate Validator

    deactivate User
```

## Reader

The reader is responsible for accepting user input through either of the
following means:

1. Directly passed as a CLI flag,
2. Read from STDIN where the data is piped into it.
3. Read the `$GIT_DIR/COMMIT_EDITMSG` file.

In other words, Crisp can read a commit message if it is directly passed to it
as an argument of the `message` command as shown below:

```console
crisp message "$(git show --no-patch --format=%B)"
```

While that is not the recommended approach to lint a commit message, it exists
for quickly validating a single message. The recommended approach to lint a
commit message is to pass the data to Crisp through `STDIN`. Crisp will read
from `STDIN` if the `--stdin` flag is passed to the `message` command.

Piping a commit message to Crisp is possible like this:

```console
git show --no-patch --format=%B | crisp message --stdin
```

In case no commit message was piped in to Crisp, the reader will fallback to
reading from the
[`$GIT_DIR/COMMIT_EDITMSG`](https://git-scm.com/docs/git-commit#Documentation/git-commit.txt-codeGITDIRCOMMITEDITMSGcode)
file.

The diagram below provides a better understanding of how the reader works
behind the scenes.

```mermaid
stateDiagram-v2
    state fork_state <<fork>>
    state if_stdin <<choice>>

    state "Read from CLI argument" as s1
    state "Read from STDIN" as s2
    state "Read from pipe" as s3
    state "Read from file" as s4

    [*] --> fork_state
    fork_state --> s1
    fork_state --> s2

    s1 --> Parser
    s2 --> if_stdin

    if_stdin --> s3 : if content is piped
    if_stdin --> s4 : if content is not piped

    s3 --> Parser
    s4 --> Parser

    Parser --> Validator
    Validator --> [*]
```

## Parser

The parser is responsible for receiving content from the reader and parsing it
into the following components:

1. **Header** — the first line of the commit message, containing the type,
   optional scope, and description.
2. **Body** — optional free-form text separated from the header by a blank line.
3. **Footers** — optional key-value pairs (e.g. `BREAKING CHANGE`, `Closes`,
   `Fixes`, `Refs`) appearing after the body.

Upon a successful parse, the logic returns a `CommitMessage` struct for further
processing and validation:

```go
type CommitMessage struct {
    Type        string
    Scope       string
    Description string
    Body        string
    Footers     map[string]string
}
```

### Header parsing

The header is parsed using a regular expression that expects the following
format:

```
<TYPE>(<SCOPE>): <DESCRIPTION>
```

The scope is optional. If the header does not match this format, an error is
returned immediately with a reference to the Conventional Commits specification.

### Body and footer parsing

Lines after the header are processed sequentially. The parser tracks whether it
is inside the footer section using an internal flag. A line is treated as a
footer if it matches a recognised key (`BREAKING CHANGE`, `Closes`, `Fixes`, or
`Refs`) followed by a colon and a value. Once a footer line is detected, all
subsequent lines are treated as footers. Body lines are accumulated until the
footer section begins.

The diagram below illustrates the full parsing flow:

```mermaid
flowchart TD
    A[Raw commit message string] --> B[Split into lines]
    B --> C{First line empty?}
    C -- Yes --> D[Return error: no commit message]
    C -- No --> E[Parse header with regex]
    E --> F{Header matches format?}
    F -- No --> G[Return error: invalid format]
    F -- Yes --> H[Extract Type, Scope, Description]
    H --> I[Process remaining lines]
    I --> J{Line is a known footer?}
    J -- Yes --> K[Add to footers map\nSet inFooter = true]
    J -- No --> L{inFooter = true?}
    L -- Yes --> M[Discard line]
    L -- No --> N[Append to body]
    K --> I
    M --> I
    N --> I
    I --> O[Return CommitMessage struct]
```

## Validator

After Crisp has parsed and generated a "commit message" object, the validator
can receive the said object for further processing and validation. The validator
performs the following operations:

1. Checks whether the commit type is in accordance to the list of accepted
   keywords (`build`, `ci`, `docs`, `feat`, `fix`, `perf`, `refactor`, `style`,
   `test`, `chore`).

2. If the optional commit message scope is provided, check whether it is
   lower-cased and contains valid characters.

3. Checks whether the commit message subject is provided and does not end with a
   period (`.`) nor starts with a capitalised letter.

4. Checks whether the length of the commit message does not exceed more than 50
   characters long.

<!--prettier-ignore-start-->
:::note
A future update will enable the functionality to provide a user-defined list of
accepted keywords, for now they are hardcoded and based on the commit messages
standards used by the Angular project ([see the docs](https://github.com/angular/angular/blob/22b96b9/CONTRIBUTING.md#type)).
Along with the ability to configure the list of accept commit types, users will also
be able to define the allowed length of the commit message.
:::
<!--prettier-ignore-end-->

On successful validation, Crisp returns an appropriate `valid commit message`
prompt for the user with an exit status code of `0`.
