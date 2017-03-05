# Bodyguard

Bodyguard protects the context boundaries of your application. 

Its authorization rules, called policies, are composed of plain modules and functions, so they can be leveraged in controllers, sockets, views, and contexts.

Bodyguard also provides the means to execute authorized actions in a composable way.

Please refer to [the complete documentation](https://hexdocs.pm/bodyguard/) for details beyond this README.

Version 2.x is quite different from previous versions, so refer to [the *1.x* branch](https://github.com/schrockwell/bodyguard/tree/1.x) if you are using versions *0.x* or *1.x*.

* [Hex](https://hex.pm/packages/bodyguard)
* [GitHub](https://github.com/schrockwell/bodyguard)
* [Docs](https://hexdocs.pm/bodyguard/)

## Quick Example

```elixir
defmodule MyApp.Blog do
  use Bodyguard.Policy

  def authorize(user, :update_post, %{post: post}) do
    # Return :ok to permit
    # Return {:error, reason} to deny
  end
end

# In a controller, authorize an action
with :ok <- MyApp.Blog.authorize(user, :update_post, %{post: post}) do
  # ...
end
```

## Authorization

To implement the `Bodyguard.Policy` behaviour, define `authorize(user, action, params)` callbacks, which must return:

* `:ok` to permit the action, or
* `{:error, reason}` to deny the action (most commonly `{:error, :unauthorized}`)

```elixir
defmodule MyApp.Blog do
  use Bodyguard.Policy

  # Admin users can do anything
  def authorize(%Blog.User{role: :admin}, _, _), do: :ok

  # Regular users can create posts
  def authorize(_, :create_post, _), do: :ok

  # Regular users can modify their own posts
  def authorize(user, action, %{post: post}) when action in [:update_post, :delete_post] 
    and user.id == post.user_id, do: :ok

  # Catch-all: deny everything else
  def authorize(_, _, _), do: {:error, :unauthorized}
end
```

The `use` macro also injects `authorize!/3` (raises on failure) and `authorize?/3` (returns a boolean) wrapper functions for convenience.

For more details, see `Bodyguard.Policy` in the docs.

## Controller Actions

Phoenix 1.3 introduces the `action_fallback` controller macro. This is the recommended way to deal with authorization failures.

The fallback controller should handle any `{:error, reason}` results returned by `authorize/3` callbacks.

```elixir
defmodule MyApp.Web.FallbackController do
  use MyApp.Web, :controller

  def call(conn, {:error, :unauthorized}) do
    conn
    |> put_status(:forbidden)
    |> render(MyApp.Web.ErrorView, :"403")
  end
end

defmodule MyApp.Web.PostController do
  use MyApp.Web, :controller
  alias MyApp.Blog

  action_fallback MyApp.Web.FallbackController

  def index(conn, _) do
    user = # get current user
    with :ok <- Blog.authorize(user, :list_posts) do
      posts = Blog.list_posts(user)
      render(conn, posts: posts)
    end
  end
end
```

If you wish to deny access without leaking the existence of a particular resource, consider returning `{:error, :not_found}` and handle it appropriately in the fallback controller.

## Composable Actions

The concept of an authorized action is encapsulated by a `Bodyguard.Action` struct. It can be constructed with some defaults, modified during the request cycle, and finally executed in a controller or socket action.

This example is exactly equivalent to the above:

```elixir
defmodule MyApp.Web.PostController do
  use MyApp.Web, :controller
  import Bodyguard.Action
  alias MyApp.Blog

  action_fallback MyApp.Web.FallbackController
  
  def index(conn, _) do
    user = # get current user

    act(Blog)                   # Initializes a %Bodyguard.Action{}
    |> put_user(user)           # Assigns the user
    |> authorize(:list_posts)   # Calls Blog.authorize/3 callback
    |> run(fn action ->         # Only executed if authorization passes
      posts = Blog.list_posts(action.user)
      render(conn, posts: posts)
    end)                        # Job is returned – a rendered conn
  end
end
```

The function passed to `run/2` is called the *job*, and it only executes if authorization succeeds. If not, then the job is skipped, and the result of the authorization failure is returned, to be handled by the fallback controller.

This particular example is verbose for demonstration, but parameters common to each controller action can be constructed ahead of time via a `Bodyguard.Plug.BuildAction` plug.

There are many more options – see `Bodyguard.Action` in the docs for details.

## Plugs

* `Bodyguard.Plug.BuildAction` – create an Action with some defaults on the connection
* `Bodyguard.Plug.Authorize` – perform authorization in the middle of a pipeline

## Installation

  1. Add `bodyguard` to your list of dependencies in `mix.exs`.

    ```elixir
    def deps do
      [{:bodyguard, "~> 2.0.0"}]
    end
    ```

  2. Create an error view for handling `403 Forbidden`.

    ```elixir
    defmodule MyApp.ErrorView do
      use MyApp.Web, :view

      def render("403.html", _assigns) do
        "Forbidden"
      end
    end
    ```

  3. Wire up a [fallback controller](#controller-actions) to render this view on authorization failures.

  4. Add `use Bodyguard.Policy` to contexts that require authorization, and implement the `authorize/3` callback.

## Alternatives

Not what you're looking for?

* [PolicyWonk](https://github.com/boydm/policy_wonk)
* [Canada](https://github.com/jarednorman/canada)
* [Canary](https://github.com/cpjk/canary)

## License

MIT License, Copyright (c) 2017 Rockwell Schrock

## Acknowledgements

Thanks to [Ben Cates](https://github.com/bencates) for helping maintain and mature this library.
