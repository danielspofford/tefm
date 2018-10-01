# Umbrella projects

Umbrella projects are opionated mono repos.

## When should I use them?

When you have a group of interdependent applications that share a rolled-up
config and dependencies without conflict.

## Why should I use them?

- To make it convenient to compile, test, and run multiple applications at once.

- To highlight the boundary between functionality.

A common example is a business logic application alongside a web application
that exposes an API.

## Alternatives

### Local dependencies

Use a mono repo pattern while using the [path](path_opts) options when
specifying dependencies within the repo. Projects that use this approach are
sometimes referred to as [Poncho projects](poncho_projects).

### Remote dependencies

Use a traditional stand alone repo while using [Hex](hex_opts) or
[GitHub](git_opts) options to resolve dependencies. This is sometimes slower to
iterate with if you are regularly having to update a Hex package. In such cases
it is instead recommended to, while developing, point the dependency at a
particular Git commit or branch. Take care to revert that change and understand
version dependencies between applications before deploying.

[poncho_projects]: https://embedded-elixir.com/post/2017-05-19-poncho-projects/
"Poncho projects"
[git_opts]: https://hexdocs.pm/mix/Mix.Tasks.Deps.html#module-git-options-git
"Git options"
[path_opts]: https://hexdocs.pm/mix/Mix.Tasks.Deps.html#module-path-options-path
"Path options"
[hex_opts]: https://hexdocs.pm/mix/Mix.Tasks.Deps.html#module-hex-options-hex
"Hex options"
