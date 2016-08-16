Plug生活在Phoenix的HTTP层的心脏中，而且Phoenix将Plug放在首要位置。我们的连接循环中的每一步都离不开plugs，Phoenix的核心组件，类似Endpoints，Routers，和控制器，本质上都是Plugs。让我们来看看是什么使得Plug如此特别。

[Plug](https://github.com/elixir-lang/plug) 特指web应用中可组合的模块。它也是不同web服务的连接适配器的抽象层面。Plug的基本思想就是统一我们所操作的"connection"概念。它与其它的HTTP中间层，类似Rack，的不同在于请求和回应在中间堆栈里是分开的。

## Plug 的规范

Plug可以简单地分为 *函数plugs* 和 *模块plugs*

### 函数Plugs

为了成为一个plug，一个函数需要简单地接受一个连接结构（`%Plug.Conn{}`）和选项。它的返回值也应是连接结构。任何符合这些标准的函数都是plug。请看例子。

```elixir
def put_headers(conn, key_values) do
  Enum.reduce key_values, conn, fn {k, v}, conn ->
    Plug.Conn.put_resp_header(conn, to_string(k), v)
  end
end
```

很简单，不是吗？

这就是我们如何使用它们来构建一系列的对Phoenix中连接的变形：

```elixir
defmodule HelloPhoenix.MessageController do
  use HelloPhoenix.Web, :controller

  plug :put_headers, %{content_encoding: "gzip", cache_control: "max-age=3600"}
  plug :put_layout, "bare.html"

  ...
end
```

通过遵守plug协议，`put_headers/2`, `put_layout/2`, 还有 `action/2`讲一个应用请求转换成了一系列明确地变形。还没完。为了更好地显示Plug的高效，让我们假设有一个脚本，需要我们检查一系列的情况，然后进行重定向，若情况失败就要停止它。如果没有plug，我们可能需要这样：

```elixir
defmodule HelloPhoenix.MessageController do
  use HelloPhoenix.Web, :controller

  def show(conn, params) do
    case authenticate(conn) do
      {:ok, user} ->
        case find_message(params["id"]) do
          nil ->
            conn |> put_flash(:info, "That message wasn't found") |> redirect(to: "/")
          message ->
            case authorize_message(conn, params["id"]) do
              :ok ->
                render conn, :show, page: find_message(params["id"])
              :error ->
                conn |> put_flash(:info, "You can't access that page") |> redirect(to: "/")
            end
        end
      :error ->
        conn |> put_flash(:info, "You must be logged in") |> redirect(to: "/")
    end
  end
end
```

注意到那几个所需的认证和授权的步骤完全被嵌套和重复了吗？让我们使用几个plugs来改进它。

```elixir
defmodule HelloPhoenix.MessageController do
  use HelloPhoenix.Web, :controller

  plug :authenticate
  plug :find_message
  plug :authorize_message

  def show(conn, params) do
    render conn, :show, page: find_message(params["id"])
  end

  defp authenticate(conn, _) do
    case Authenticator.find_user(conn) do
      {:ok, user} ->
        assign(conn, :user, user)
      :error ->
        conn |> put_flash(:info, "You must be logged in") |> redirect(to: "/") |> halt
    end
  end

  defp find_message(conn, _) do
    case find_message(conn.params["id"]) do
      nil ->
        conn |> put_flash(:info, "That message wasn't found") |> redirect(to: "/") |> halt
      message ->
        assign(conn, :message, message)
    end
  end

  defp authorize_message(conn, _) do
    if Authorizer.can_access?(conn.assigns[:user], conn.assigns[:message]) do
      conn
    else
      conn |> put_flash(:info, "You can't access that page") |> redirect(to: "/") |> halt
    end
  end
end
```

通过将嵌套的代码块修改为一系列平行的plug变换，我们可以以更加可组合的，清晰的，和可复用的方式来实现同样的功能。

现在让我们来看看另一种plugs，模块plugs。

### 模块Plugs

模块plugs是一种能够在模块中定义connection变换的Plug。模块只需要实现两个函数：

- `init/1` 它初始化了任何要被传送给`call/2`的参数或选项
- `call/2` 它执行了connection变换。`call/2`就是我们刚才看到的函数plug

让我们来编写一个模块plug。它的功能是把`:local`键和值放到connection中，这个模块plug可以在其他plugs，控制器动作，和views中使用。

```elixir
defmodule HelloPhoenix.Plugs.Locale do
  import Plug.Conn

  @locales ["en", "fr", "de"]

  def init(default), do: default

  def call(%Plug.Conn{params: %{"locale" => loc}} = conn, _default) when loc in @locales do
    assign(conn, :locale, loc)
  end
  def call(conn, default), do: assign(conn, :locale, default)
end

defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
    plug HelloPhoenix.Plugs.Locale, "en"
  end
  ...
```

我们可以通过`plug HelloPhoenix.Plugs.Locale, "en"`来讲这个模块plug添加到我们的browser管道。在`init/1`回调中，我们传送了一个默认的locale，如果params中没有东西，就会使用它。我们也使用了模式匹配来定义多重`call/2`函数，目的是验证locale是否在params中，如果没有匹配到，就会返回“en”。

本章全部都在讲Plug。Phoenix在整个stack中，都贯彻着plug这种可组合的变换的设计。这只是开始，如果你问我们，“我能把这个放在一个plug中吗？”，答案通常是“Yes！”。
