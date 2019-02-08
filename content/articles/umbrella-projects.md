# Umbrella projects

Umbrella projects are a means of dealing with internal dependencies.

[Dependencies and umbrella projects](internal_deps):

> Internal dependencies are the ones that are specific to your project. They
> usually don’t make sense outside the scope of your
> project/company/organization. Most of the time, you want to keep them private,
> whether due to technical, economic or business reasons.

> If you have an internal dependency, Mix supports two methods to work with
> them: Git repositories or umbrella projects.

[Don't drink the kool aid](koolaid):

> Umbrella projects are a convenience to help you organize and manage multiple
> applications. While it provides a degree of separation between applications,
> those applications are not fully decoupled, as they are assumed to share the
> same configuration and the same dependencies.

> The pattern of keeping multiple applications in the same repository is known
> as “mono-repo”. Umbrella projects maximize this pattern by providing
> conveniences to compile, test and run multiple applications at once.

> If you find yourself in a position where you want to use different
> configurations in each application for the same dependency or use different
> dependency versions, then it is likely your codebase has grown beyond what
> umbrellas can provide.

## Alternatives

### Local dependencies

Use a mono repo pattern while using the [path](path_opts) options when
specifying dependencies within the repo. Projects that use this approach are
sometimes referred to as [Poncho projects](poncho_projects).

### Remote dependencies

Use a traditional stand alone repo while using the [Hex](hex_opts) or
[GitHub](git_opts) options to resolve dependencies. Keep in mind Hex
dependencies can be prohibitively slow to use within this context if you are
making frequent changes to the main project as well as the dependency.

[koolaid]: https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-projects.html#dont-drink-the-kool-aid
[internal_deps]: https://elixir-lang.org/getting-started/mix-otp/dependencies-and-umbrella-projects.html#internal-dependencies
[poncho_projects]: https://embedded-elixir.com/post/2017-05-19-poncho-projects/
"Poncho projects"
[git_opts]: https://hexdocs.pm/mix/Mix.Tasks.Deps.html#module-git-options-git
"Git options"
[path_opts]: https://hexdocs.pm/mix/Mix.Tasks.Deps.html#module-path-options-path
"Path options"
[hex_opts]: https://hexdocs.pm/mix/Mix.Tasks.Deps.html#module-hex-options-hex
"Hex options"
