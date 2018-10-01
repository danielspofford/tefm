# Generate a project

The two most common ways to generate an Elixir project are `mix new` and `mix
phx.new`. You will note both commands support an `--umbrella` option to generate
an umbrella project. See [Umbrella Projects](umbrella_projects) for more
information.

## Mix new

This is the purest choice and makes sense if you don't plan on using Phoenix,
more on that in the next section.

```
$ mix new my_project_name
```

Checkout `$ mix help new` for complete documentation. Noteworthy options:

- `--sup`

- `--umbrella`

## Phoenix new

Phoenix should be used if you need templating or value the consistency of a
framework. Phoenix presence and pubsub can be used via the
[phoenix_pubsub](phoenix_pubsub) dependency without having to bring in
[phoenix](phoenix) itself.


```
$ mix phx.new my_project_name
```

Checkout `$ mix help phx.new` for complete documentation. Noteworthy options:

- `--no-webpack`

- `--no-ecto`

- `--no-html`

- `--binary-id`

- `--umbrella`

[umbrella_projects]: content/articles/umbrella-projects.md "Umbrella Projects"
[phoenix]: https://github.com/phoenixframework/phoenix "Phoenix"
[phoenix_pubsub]: https://github.com/phoenixframework/phoenix_pubsub "Phoenix
PubSub"
