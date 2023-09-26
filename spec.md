# Credential Helpers Specification

The Credential Helper standard defines an extensible protocol for fetching
credentials (e.g., for RPCs) from a subprocess.


## Authors

  - Yannic Bonenberger (EngFlow)
  - Tiago Quelhas (Google)


## Requirements

The key words **MUST**, **MUST NOT**, **REQUIRED**, **SHALL**, **SHALL NOT**,
**SHOULD**, **SHOULD NOT**, **RECOMMENDED**,  **MAY**, and **OPTIONAL** in this
document are to be interpreted as described in
[RFC 2119](https://www.ietf.org/rfc/rfc2119.txt).

An implementation is not compliant if it fails to satisfy one or more of the
**MUST** or **REQUIRED** level requirements of this specification. An
implementation that satisfies all the **MUST** or **REQUIRED** level and all the
**SHOULD** level requirements is said to be *unconditionally compliant*; one
that satisfies all the **MUST** level requirements but not all the **SHOULD**
level requirements for its protocols is said to be *conditionally compliant*.


## Terminology

<!-- Keep sorted! -->

  - `EBNF`: [Extended Backus–Naur/Backus-Normal Form](https://en.wikipedia.org/wiki/Extended_Backus–Naur_form#Table_of_symbols);
    a syntax notation for context-free grammars.

  - `JSON`: a data interchange format specified in
    [RFC 8259](https://www.ietf.org/rfc/rfc8259.txt).

  - `JSON-Schema`: a specification for
    [schemas of JSON data](https://json-schema.org/).
  
  - `exit status`: the
    [status code a program exits with](https://en.wikipedia.org/wiki/Exit_status).
  
  - `stderr`: a stream to which which a program
    [writes its errors](https://en.wikipedia.org/wiki/Standard_streams#Standard_error_\(stderr\)).
  
  - `stdin`: a stream from which a program
    [reads its input data](https://en.wikipedia.org/wiki/Standard_streams#Standard_input_\(stdin\)).
  
  - `stdout`: a stream to which a program
    [writes its output data](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_\(stdout\)).


## Introduction

A Credential Helper is an program that handles storage for credentials and is
invoked by a Tool needing credentials. A Credential Helper **MUST** be
executable directly on the host operating system.

Credential Helper invocations can either succeed or fail. If an invocation
succeeds, the Credential Helper **MUST** terminate with an exit status
indicating success on the host operating system (typically `0`). If an
invocation fails, the Credentail Helper **MUST** terminate with an exit status
that indicates failure (typically any value other than `0`). A Credential Helper
**MAY** indicate varying types of failures by setting the exit status to
different values when terminating. A Credential Helper **MAY** also provide
additional context about the failure on its stderr. Tools **SHOULD NOT** rely on
the semantics of a Credential Helper's exit status beyond  whether an invocation
was successful or rely on any messages written on stderr.

To invoke a Credential Helper, a Tool specify a list of command-line arguments
for the subprocess. In this specification, we will assume that command-line
arguments are specified as a list of strings called `args`.

> The actual format of command-line arguments depends on the platform Tool and
> Credential Helper run on. For example, some platforms provide command-line
> arguments as list of strings `argv`, including, by convention, the
> executed binary's path as first element `argv[0]` while `args` used in this
> specification omits this parameter.
>
> Example:
>
> Assuming we have the command `echo foo bar`, `argv = ["echo", "foo", "bar"]`
> on most platforms while `args = ["foo", "bar"]`.

If the list of command-line arguments `args` is not empty, the first argument
is known as `command = args[0]` of the Credential helper. Otherwise, if the list
of command-line arguments is empty and thus the command is unspecified, the
Credential Helper **SHOULD** print usage instructions and indicate that the
invocation failed.

The list containing all command-line arguments following the command is known as
`parameters = args[1...]` and passed to the command.

In addition to the list of parameters, commands **MAY** have required or
optional `input`. If a command specifies such an input, a Tool **MUST** pass it
by writing to the Credential Helper's stdin using JSON encoding. Commands
that have a required or optional input **MUST** specify a JSON schema for the
input.

Commands **MAY** specify an `output`. If a command does so, the Credential
Helper **MUST** write it to their stdout using JSON encoding. Commands
specifying outputs **MUST** specify a JSON schema for the output.


## Commands

This section specifies the list of commands a Credential Helper **MUST** and
**SHOULD** implement. All commands **MUST** conform to the following grammar in
EBNF. A Credential Helper **MAY** extend the list of *standard-command* specified
below with *user-defined* *extension-command*s.


```
<command>               ::= <standard-command> | <extension-command>
<standard-command>      ::= <identifier>
<extension-command>     ::= <identifier> "-" <identifier> { "-" <identifier> }
<identifier>            ::= <letter> <character> <character> { <character> }
<character>             ::= <letter> | <digit>
<letter>                ::= "a" | "b" | "c" | "d" | "e" | "f" | "g" | "h" | "i" | "j" | "k" | "l" | "m" | "n" | "o" | "p" | "q" | "r" | "s" | "t" | "u" | "v" | "w" | "x" | "y" | "z"
<digit>                 ::= "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9"
```

In other words, a command is either a *standard-command* or an
*extension-command*. A *standard-command* consists of a single *identifier*
while an *extension-command* consists of two or more *identifier*s joined by a
single `-`. *identifier*s contains only lower-case (ASCII) letters and numbers,
starts with a letter, and is of length three or more.

Examples:

  - *standard-command*:
    - `get`
    - `store`
    - `login`

  - *extension-command*:
    - `get-credentials`
    - `store-credentials-locally`
    - `cluster-interactive-browser-login`

<!--
  Spec requirement: all commands below **MUST** be `<standard-command>`.
-->

### `get`

The `get` command is used by tools to retrieve credentials from a Credential
Helper. A Credential Helper **MUST** implement this command.

A Tool **SHOULD** cache credentials to avoid invoking Credential Helpers
unneccessarily often.

#### Parameters

`get` does not require any *command-specific* parameters. A credential Helper
**MAY** define additional parameters for `get` (e.g., flags for printing debug
information), but **MUST NOT** require them. A credential Helper **MAY** ignore
any additional parameters or return an error. A Tool **SHOULD NOT** provide
additional parameters when invoking a Credential Helper's `get` command.

#### Input

A Tool invoking a Credential Helper **MUST** provide an input conforming to the
schema defined in
[GetCredentialsRequest](schemas/get-credentials-request.schema.json) which
**SHOULD NOT** contain addtional properties. A Tool **MUST** ignore failures
providing the input to the Credential Helpers.

A Credential Helper **MAY** read the input provided by the tool to decide which,
if any, credentials to return as output. Credential Helpers **SHOULD** not fail
if the input contains additional properties.

<details>
  <summary>
    Examples
  </summary>

  ```json
  {
    "uri": "grpcs://example.com/com.example.package.Service/Method"
  }
  ```

  ```json
  {
    "uri": "https://api.example.org/rest/endpoint"
  }
  ```
</details>

#### Output

A Credential Helper **MUST** return an output conforming to the schema define in
[GetCredentialsResponse](schemas/get-credentials-response.schema.json) which
**SHOULD NOT** contain additional properties. When seeing unsupported input
(e.g., an unexpected protocol or domain of the provided URI), the Credential
Helper **SHOULD** return an error indicating the unsupported input instead of
returning an output.

A Tool invoking a Credential Helper **SHOULD NOT** fail if the output contains
additional properties.

<details>
  <summary>
    Examples
  </summary>

  Empty response indicating that no credentials are needed.

  ```json
  {
    "headers": {}
  }
  ```

  This is equivalent to the following two responses:

  ```json
  {}
  ```

  ```json
  {
    "headers": {
      "header1": []
    }
  }
  ```

  Attach an [Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization)
  header (e.g., with a [JWT](https://jwt.io/)).

  ```json
  {
    "headers": {
      "Authorization": ["Bearer <token...>"]
    }
  }
  ```

  Attach multiple custom headers for authentication.

  ```json
  {
    "headers": {
      "x-custom-auth-type": ["proprietary-auth"],
      "x-custom-auth-token": ["<token...>"]
    }
  }
  ```
</details>


## Appendix

Credential Helpers were previously specified in
[Credential Helpers for Bazel](https://github.com/bazelbuild/proposals/blob/main/designs/2022-06-07-bazel-credential-helpers.md).

A reference implementation for Tools invoking Credential Helpers can be found in
[Bazel](https://cs.opensource.google/bazel/bazel/+/master:src/main/java/com/google/devtools/build/lib/authandtls/credentialhelper/;drc=845fc89e503c0ec84ba03b29dafa7c65e6051c9b).
