# Code Climate Engine specification

**Note: This specification is a draft. We welcome your suggestions for improvement, and there may be bumps along the way!**

## Overview

A Code Climate engine is a standalone program which accepts a configuration and source code, and returns static analysis results. It can be implemented in any programming language, and is distributed as a Docker image.

## Input

The Engine Docker container is provided the source code to analyze at `/code`, which is mounted as a read-only volume.

* `/code`, a directory containing the files to analyze (read-only)

Engines accept a configuration in JSON format passed as a read-only file mounted at `/config.json`.

Engines can define their own appropriate configuration keys and values, based on
their needs. A developer invoking a Code Climate engine stores their configuration
for all engines in a single `.codeclimate.yml` file. The Code Climate CLI (and other tools compatible with Code Climate Engines) parse and interpret the YAML file to produce a single, simple JSON config object for each engine.

Certain keys of the config object will be passed to all engines, and must be respected in order to conform with the specification. Today, there is only one such key:

* `exclude_paths` -- An Array of file paths (relative to the `/code` directory) that should be ignored for the purposes of analysis. No Issues should be emitted for excluded files.

## Output

Engines stream static analysis Issues to STDOUT in JSON format. When possible, results should be emitted as soon as they are computed (streaming, not buffered). Documents are terminated by the [null character][null] (`\0` in most programming languages), but can additionally be separated by newlines. See below forth

Unstructured information can be printed on STDERR for the purposes of aiding debugging.

An engine must exit with a zero exit code to be considered a success. Any nonzero exit code indicates a fatal error in the static analysis and the results for the entire analysis will be discarded (even if some were previously emitted).

## Data Types

### Issues

An `issue` represents a single instance of a real or potential code problem, detected by a static analysis Engine.

```json
{
  "type": "issue",
  "check_name": "Bug Risk/Unused Variable",
  "description": "Unused local variable `foo`",
  "categories": ["Complexity"],
  "location": Location,
  "remediation_points": 500
}
```

* `type` -- Required. Must always be "issue".
* `check_name` -- Required. A unique name representing the static analysis check that emitted this issue. `check_name`
* `description` -- Required. A string explaining the issue that was detected.
* `categories` -- Required. At least one category indicating the nature of the issue being reported.
* `location` -- Required. A Location object representing the place in the source code where the issue was discovered.
* `remediation_points` -- Optional. An abstract, relative integer indicating a rough estimate of how long it would take to resolve the reported issue.

#### Descriptions

Descriptions must be a single line of text (no newlines), with no HTML formatting contained within. Ideally, descriptions should be fewer than 70 characters long, but this is not a requirement.

Descriptions support one type of basic Markdown formatting, which is the use of backticks to produce inline &lt;code&gt; tags that are rendered in a fixed width font. Identifiers like class, method and variable names should be wrapped within backticks whenever possible for optimal rendering by tools that consume Engines data.

#### Categories

Issues must be associated with one or more categories. Valid issue `categories` are:

- `Bug Risk` -- TODO describe me
- `Clarity` -- TODO describe me
- `Compatibility` -- TODO describe me
- `Complexity` -- TODO describe me
- `Duplication` -- TODO describe me
- `Security` -- TODO describe me
- `Style` -- TODO describe me

#### Remediation Points

Remediation points are an abstract, relative scale to express the estimated time it would take for a developer to resolve an issue. They are abstract because they do not map directly to absolute time durations like minutes and hours. Providing remediation points is optional, but they can be useful to certain tools that consume Engines data and generate reports related to the level of effort required to improve a codebase (like CodeClimate.com).

Here are some guidelines to compute appropriate remediation points values for an Issue:

* The more local an issue is, generally the easier it is to fix. For example, issues that only require consideration of a single line tend to be easier to fix than issues that potentially affect an entire function, class, module, or program.
* TODO more

The baseline remediation points value is 50,000, which is the time it takes to fix a trivial code style issue like a missing semicolon on a single line, including the time for the developer to open the code, make the change, and confidently commit the fix. All other remediation points values are expressed in multiples of that Basic Remediation Point Value.

### Locations

Locations refer to ranges of a source code file. All locations are expressed as ranges, and therefore have a beginning and an end (which can be the same).

A Location has one of two formats:

```
{
  "path": "path/to/file.css",
  "lines": {
    "begin: 13,
    "end": 14
  }
}
```

Or:

```
{
  "path": "path/to/file.css",
  "positions": {
    "begin: Position,
    "end": Position
  }
}
```

All Locations require a `path` property, which is the file path relative to `/code`.

Locations of the first form (_line-based_ locations) emit a beginning and end line number for the issue, which form a range. Line numbers are 1-based, so the first line of a file would be represented by `1`. Line ranges are evaluated inclusively, so a range of `{"begin": 9, "end": 11}` would represent lines 9, 10 and 11.

Locations in the second form (_position-based_ locations) allow more precision by including refrences to the specific characters that form the source code range representing the issue.

#### Positions

Positions refer to specific characters within a source file, and can be expressed in two ways:

1. Line and column coordinates. (You can roughly think of these as X/Y axis.)
2. Absolute character offsets, for the _entire source buffer_.

For example:

```
{
  line: 3,
  column: 10
}
```

Or:

```
{
  offset: 4
}
```

Line and column numbers are 1-based. Therefore,
a Position of `{ "line": 2, "column": 3 }` represents the third character on the second
line of the file.

Offsets, however are 0-based. A Position of `{ "offset": 4 }` represents the _fifth_ character in the file. Importantly, the `offset` is from the beginning of the file, not the beginning of a line. Newline characters (and all characters) count when computing an offset.

## Packaging

Engines are packaged and distributed as Docker images, which allows them to be
easily portable between systems, regardless of the programming language they are
implemented in. We recommend Engine implementors use a `Dockerfile` to automate
the builds of these images. The `Dockerfile` must follow these specifications:

* The images must specify a `MAINTAINER`.
* The `/code` must be declared as a `VOLUME`.
* The `WORKDIR` must be specified as `/code`
* A non-root user named `app` must be created with UID and GID 9000 and declared
  using the `USER` directive.
* A default command (`CMD`) must be declared so that the engine can be invoked without specifying a command
* The image must not include any `EXPOSE` directives
* The image must not include any `ONBUILD` directives

Here is an example of a `Dockerfile` that follows the specifications for a Go program:

```
TODO Dockerfile EXAMPLE Here
```

## Resource Restrictions

In order to ensure analysis runs reliably across a variety of systems, Engines
must conform to some basic resource restrictions:

* The Docker image for an Engine must not exceed 250MB, including all layers
* The combined total RSS memory usage my all processes within the Docker container must not exceed 256MB at any time.
* All Engines must complete and exit within 10 minutes.

## Security Restrictions

Engines run in a secured runtime environment, within container-based virtualization
provided by Docker.

* The root filesystem (`/`) is mounted read-only. A `/tmp` volume is mounted read-write for temporary file storage during the engine run.
* Engines run with no network access (`--net=none` in Docker). They must not rely on making any external network calls.
* Engines run with the minimal set of Linux capabilities (`--cap-drop all` in Docker)
* Engines are always ran as a user `app` with UID and GID of 9000, and never `root`.

[null]: http://en.wikipedia.org/wiki/Null_character
