## The Spec

### Tasks

The core of every `tusk.yml` file is a list of tasks. Tasks are declared at the
top level of the `tusk.yml` file and include a list of tasks.

For the following tasks:

```yaml
tasks:
  hello:
    run: echo "Hello, world!"
  goodbye:
    run: echo "Goodbye, world!"
```

The commands can be run with no additional configuration:

```text
$ tusk hello
Running: echo "Hello, world!"
Hello, world!
```

Tasks can be documented with a one-line `usage` string and a slightly longer
`description`. This information will be displayed in help messages:

```yaml
tasks:
  hello:
    usage: Say hello to the world
    description: |
      This command will echo "Hello, world!" to the user. There's no
      surprises here.
    run: echo "Hello, world!"
  goodbye:
    run: echo "Goodbye, world!"
```

### Run

The behavior of a task is defined in its `run` clause. A `run` clause can be
used for commands, sub-tasks, or setting environment variables. Although each
`run` item can only perform one of these actions, they can be run in succession
to handle complex scenarios.

#### Command

In its simplest form, `run` can be given a string or list of strings to be
executed serially as shell commands:

```yaml
tasks:
  hello:
    run: echo "Hello!"
```

This is a shorthand syntax for the following:

```yaml
tasks:
  hello:
    run:
      - command: echo "Hello!"
```

If any of the run commands execute with a non-zero exit code, Tusk will
immediately exit with the same exit code without executing any other commands.

For executing shell commands, the interpreter used will be the value of the
`SHELL` environment variable. If no environment variable is set, the default is
`sh`.

#### Set Environment

The second type of action a `run` clause can perform is setting or unsetting
environment variables. To do so, simply define a map of environment variable
names to their desired values:

```yaml
tasks:
  hello:
    options:
      proxy-url:
        default: http://proxy.example.com
    run:
      - set-environment:
          http_proxy: ${proxy-url}
          https_proxy: ${proxy-url}
          no_proxy: ~
      - command: curl http://example.com
```

Passing `~` or `null` to an environment variable will explicitly unset it, while
passing an empty string will set it to an empty string.

#### Sub-Tasks

Run can also execute previously-defined tasks:

```yaml
tasks:
  one:
    run: echo "Inside one"
  two:
    run:
      - task: one
      - command: echo "Inside two"
```

For any option that a sub-task defines, the parent task can pass a value, which
is treated the same way as passing by command-line would be. To do so, use the
long definition of a sub-task:

```yaml
tasks:
  greet:
    options:
      person:
        default: World
    run: echo "Hello, ${person}!"
  greet-myself:
    run:
      task:
        name: greet
        options:
          person: me
```

In cases where a sub-task may not be useful on its own, define it as private to
prevent it from being invoked directly from the command-line. For example:

```yaml
tasks:
  configure-environment:
    private: true
    run:
      set-environment: {APP_ENV: dev}
  serve:
    run:
      - task: configure-environment
      - command: python main.py
```

### When

For conditional execution, `when` clauses are available.

```yaml
run:
  when:
    os: linux
  command: echo "This is a linux machine"
```

In a `run` clause, any item with a true `when` clause will execute. There are
five different checks supported:

- `command` (string): Execute if the command runs with an exit code of `0`.
- `exists` (string): Execute if the file exists.
- `os` (list): Execute if the operating system matches any one from the list.
- `environment` (map[string -> string]): Execute if the environment variable
  matches the value it maps to. To check if a variable is not set, the value
  should be `~` or `null`.
- `equal` (map[string -> string]): Execute if the given option equals the value
  it maps to.
- `not-equal` (map[string -> string]): Execute if the given option does not
  equal the value it maps to.

The `when` clause supports any number of different checks as a list, where each
check must pass individually for the clause to evaluate to true. Here is a more
complicated example of how `when` can be used:

```yaml
tasks:
  echo:
    options:
      cat:
        usage: Cat a file
    run:
      - when:
          os:
            - linux
            - darwin
        command: echo "This is a unix machine"
      - when:
          - exists: my_file.txt
          - equal: {cat: true}
          - command: command -v cat
        command: cat my_file.txt
```

### Options

Tasks may have options that are passed as GNU-style flags. The following
configuration will provide `-n, --name` flags to the CLI and help documentation,
which will then be interpolated:

```yaml
tasks:
  greet:
    options:
      name:
        usage: The person to greet
        short: n
        environment: GREET_NAME
        default: World
    run: echo "Hello, ${name}!"
```

The above configuration will evaluate the value of `name` in order of highest
priority:

1. The value passed by command line flags (`-n` or `--name`)
2. The value of the environment variable (`GREET_NAME`), if set
3. The value set in default

#### Option Types

Options can be of the types `string`, `integer`, `float`, or `boolean`, using
the zero-value of that type as the default if not set. Options without types
specified are considered strings.

For boolean values, the flag should be passed by command line without any
arugments. In the following example:

```yaml
tasks:
  greet:
    options:
      loud:
        type: bool
    run:
      - when:
          equal: {loud: true}
        command: echo "HELLO!"
      - when:
          equal: {loud: false}
        command: echo "Hello."
```

The flag should be passed as such:

```bash
tusk greet --loud
```

This means that for an option that is true by default, the only way to disable
it is with the following syntax:

```bash
tusk greet --loud=false
```

Of course, options can always be defined in the reverse manner to avoid this
issue:

```yaml
options:
  no-loud:
    type: bool
```

#### Option Defaults

Much like `run` clauses accept a shorthand form, passing a string to `default`
is shorthand. The following options are exactly equivalent:

```yaml
options:
  short:
    default: foo
  long:
    default:
      - value: foo
```

A `default` clause can also register the `stdout` of a command as its value:

```yaml
options:
  os:
    default:
      command: uname -s
```

A `default` clause also accepts a list of possible values with a corresponding
`when` clause. The first `when` that evaluates to true will be used as the
default value, with an omitted `when` always considered true.

In this example, linux users will have the name `Linux User`, while the default
for all other OSes is `User`:

```yaml
options:
  name:
    default:
      - when:
          os: linux
        value: Linux User
      - value: User
```

#### Option Values

An option can specify which values are considered valid:

```yaml
options:
  number:
    default: zero
    values:
      - one
      - two
      - three
```

Any value passed by command-line flags or environment variables must be one of
the listed values. Default values, including commands, are excluded from this
requirement.

#### Required Options

Options may be required if there is no sane default value. For a required flag,
the task will not execute unless the flag is passed:

```yaml
options:
  file:
    required: true
```

A required option cannot be private or have any default values.

#### Private Options

Sometimes it may be desirable to have a variable that cannot be directly
modified through command-line flags. In this case, use the `private` option:

```yaml
options:
  user:
    private: true
    default:
      command: whoami
```

A private option will not accept environment variables or command line flags,
and it will not appear in the help documentation.

#### Shared Options

Options may also be defined at the root of the config file to be shared between
tasks:

```yaml
options:
  name:
    usage: The person to greet
    default: World

tasks:
  hello:
    run: echo "Hello, ${name}!"
  goodbye:
    run: echo "Goodbye, ${name}!"
```

Any shared variables referenced by a task will be exposed by command-line when
invoking that task. Shared variables referenced by a sub-task will be evaluated
as needed, but not exposed by command-line.

### CLI Metadata

It is also possible to create a custom CLI tool for use outside of a project's
directory by using shell aliases:

```bash
alias mycli="tusk -f /path/to/tusk.yml"
```

In that case, it may be useful to override the tool name and usage text that
are provided as part of the help documentation:

```yaml
name: mycli
usage: A custom aliased command-line application

tasks:
  ...
```

The example above will produce the following help documentation:

```text
mycli - A custom aliased command-line application

Usage:
  mycli [global options] <task> [task options]

Tasks:
  ...
```

### Interpolation

The interpolation syntax for a variable `foo` is `${foo}`, meaning any instances
of `${foo}` in the configuration file will be replaced with the value of `foo`
during execution.

Interpolation is done on a task-by-task basis, meaning options defined in one
task will not interpolate to any other tasks. For each task, interpolation is
done iteratively in the order that variables are defined, with shared variables
being evaluated first. This means that options can reference other options:

```yaml
options:
  name:
    default: World
  greeting:
    default: Hello, ${name}

tasks:
  greet:
    run: echo "${greeting}"
```

Because interpolation is not always desirable, as in the case of environment
variables, `$$` will escape to `$` and ignore interpolation. It is also
possible to use alternative syntax such as `$foo` to avoid interpolation as
well. The following two tasks will both use environment variables and not
attempt interpolation:

```yaml
tasks:
  one:
    run: Hello, $${USER}
  two:
    run: Hello, $USER
```

Interpolation works by substituting the value in the `yaml` config file, then
parsing the file after interpolation. This means that variable values with
newlines or other characters that are relevant to the `yaml` spec or the `sh`
interpreter will need to be considered by the user. This can be as simple as
using quotes when appropriate.
