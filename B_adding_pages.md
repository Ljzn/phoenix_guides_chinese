本教程的任务是添加两个新页面到我们的Phoenix应用中。一个是纯静态页面，另一个将会以URL的路径作为输入，并将其传送给一个模板来显示。这个过程中，我们将熟悉Phoenix应用的基本构成：路由，控制器，views，和模板。

当Phoenix为我们生成了一个新应用，它构建了一个这样的顶层目录结构：

```text
├── _build
├── config
├── deps
├── lib
├── priv
├── test
├── web
```

本教程中大多数工作会在`web`目录中完成，展开时是这样：

```text
├── channels
├── controllers
│   └── page_controller.ex
├── models
├── static
├── router.ex
├── templates
│   ├── layout
│   │   └── app.html.eex
│   └── page
│       └── index.html.eex
└── views
|   ├── error_view.ex
|   ├── layout_view.ex
|   └── page_view.ex
└── web.ex
```

`controllers`,`templates`,和`views`目录中的所有文件创造了我们再上一个教程中看到的`Welcone to Phoenix!`页面。我们将学会如何简单地复用其中的一些代码。出于方便，在开发环境下，`web`目录中的任何东西都会在新的web请求出现时自动重新编译。

我们应用的所有静态资源都按类型存放在`priv/static`目录 - css，images或js。我们将构建过程中需要的资源放在`web/static`中，而`app,js`/`app.css`所需的源文件存放在`priv/static`中。现在我们不会在这里做任何改动，但以后需要时我们就有个参考。

```text
priv
└── static
    └── images
        └── phoenix.png
```

```text
web
└── static
    ├── css
    |   └── app.css
    ├── js
    │   └── app.js
    └── vendor
        └── phoenix.js
```

与`web`目录不同，当有新的网络请求时，Phoenix不会重新编译`lib`中的文件。这是故意为之！`web`与`lib`的区别，为我们以不同的方式处理应用中的状态提供了方便。`web`目录中任何东西的状态都只持续一个web请求。而`lib`包含了共用模块以及任何需要在一个web请求持续时间之外管理状态的东西。

铺垫的够多了，让我们回到第一个Phoenix页面！

### 一个新路由

路由将每个HTTP verb/path对映射到controller/action对，来处理它们。Phoenix为我们的新应用在`web/router.ex`生成了一个路由文件。本节我们将在该文件中工作。

我们的“Welcome to Phoenix!”页面的路由看上去是这样。
```elixir
get "/", PageController, :index
```

让我们理解一下这个路由告诉了我们什么。访问[http://localhost:4000/](http://localhost:4000/) 提出了一个HTTP `GET`请求到根路径。所有这种请求都会由定义在`web/controllers/page_controller.ex`文件中的`HelloPhoenix.PageController`模块中的`index`函数来处理。

当访问[http://localhost:4000/hello](http://localhost:4000/hello)时，我们将要构建的页面会简单地说“Hello World, from Phoenix!”。

创造那个页面，要做的第一件事就是为它定义一个路由。让我们打开`web/router.ex`。它此时是这样：

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html"]
    plug :fetch_session
    plug :fetch_flash
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end

  pipeline :api do
    plug :accepts, ["json"]
  end

  scope "/", HelloPhoenix do
    pipe_through :browser # Use the default browser stack

    get "/", PageController, :index
  end

  # Other scopes may use custom stacks.
  # scope "/api", HelloPhoenix do
  #   pipe_through :api
  # end
end

```

现在，我们会忽略管道连线和`scope`的使用，专注于添加路由。(我们在[Routing Guide](http://www.phoenixframework.org/docs/routing)中有对这个话题进行讨论)。

让我们添加一个路由，它会将对`/hello`的`GET`请求映射到即将被创建的`HelloPhoenix.HelloController`中的`index`动作：

```elixir
get "/hello", HelloController, :index
```

我们的`router.ex`文件中的`scope "/"`块现在会是这样：

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser # Use the default browser stack

  get "/", PageController, :index
  get "/hello", HelloController, :index
end
```

### 一个新控制器

控制器是Elixir模块，而action是它们中定义的Elixir函数。action的目的是为渲染网页而搜集数据和完成任务。我们的路由决定了我们需要一个有着`index/2`action的`HelloPhoenix.HelloController`模块。

为了达到目标，让我们创建一个新的`web/controllers/hello_controller.ex`文件，并让它看上去像这样：

```elixir
defmodule HelloPhoenix.HelloController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    render conn, "index.html"
  end
end
```

对于`use HelloPhoenix.Web, :controller`，我们将在[Controllers Guide](http://www.phoenixframework.org/docs/controllers) 中讨论。现在，让我们关注`index/2`action。

所有控制器action都有两个参数。第一个是`conn`，它是一个结构，能够容纳成吨的关于请求的数据。第二个是`params`,它是请求的参项。在这里，我们没有用到`params`，为了避免编译器警告，我们在它之前添加了`_`。

这个action的核心是`render conn, "index.html"`。这告诉了Phoenix去寻找一个叫做`index.html.eex`的模板，并渲染它。Phoenix会在以我们的控制器命名的目录下寻找模板，也就是`web/templates/hello`。

> 注意：在这里也可以使用原子作为模板名，`render conn, :index`，但是会基于所接受的头文件来选择模板，例如`"index.html"`或`"index.json"`。

负责渲染的模块是views，接下来我们将做一个新的view。

### 一个新的View

Phoenix的views负责一些重要工作。它们渲染模板。它们也像一个表示层一样，接收控制器的生数据，为在模板中使用做准备。进行这种变形操作的函数应当放在view里。

例如，假设我们有一个数据结构，它用`first_name`和`last_name`来代表一个用户，而在模板中我们希望展示用户的全名。我们可以在模板中编写代码，来使那些文本框融合成一个全名，但更好的方法是，在view中编写一个实现这个功能的函数，然后在模板中调用该函数。我们会得到一个更清晰干净的模板。

为了给`HelloController`渲染模板，我们需要一个`HelloView`。这里名字是有意义的 - view和控制器的名字前部分必须匹配。让我们先创建一个，稍后我们将详细介绍views。创建`web/views/hello_view.ex`并让它看上去像这样：

```elixir
defmodule HelloPhoenix.HelloView do
  use HelloPhoenix.Web, :view
end
```

### 一个新模板

Phoenix中模板的意思就是其中的数据可以被渲染。Phoenix使用的标准模板引擎是EEx，它代表[Embedded Elixir](http://elixir-lang.org/docs/stable/eex/) 。我们所有的模板文件都有`.eex`后缀名。

模板受制于view，view受制于控制器。在实际中，意思就是我们在`web/templates`目录中要创建一个以控制器命名的目录。对于我们的hello页面，这意味着我们需要在`web/templates`下创建一个`hello`目录，然后在其中创建一个`index.html.eex`文件。

让我开始做吧。创建`web/templates/hello/index.html.eex`并使它看上去像这样：

```html
<div class="jumbotron">
  <h2>Hello World, from Phoenix!</h2>
</div>
```

现在我们有了路由，控制器，view和模板，我们应该可以打开[http://localhost:4000/hello](http://localhost:4000/hello) 欣赏来自Phoenix的问候了！(如果你关闭了服务器，可以用`mix phoenix.server`来重启。)

![Phoenix Greets Us](/images/hello-from-phoenix.png)

我们刚做的事情中有许多值得注意的有趣的地方。在做修改时我们不需要停止并重启服务器。使得，Phoenix可以热重载！而且，尽管我们的`index.html.eex`文件只有一个`div`标签，这个页面仍然得到了完整的HTML文档。我们的主页模板被渲染进了应用布局 - `web/templates/layout/app.html.eex` 。如果你打开它，你会看见其中有一行：

    <%= render @view_module, @view_template, assigns %>

就是它在HTML被发送给浏览器之前将我们的模板渲染进了布局中。

## 另一个新页面

让我们的应用变得稍稍复杂一些。我们将添加一个新页面，它会辨认URL的一部分，并将其标记为“信使”并通过控制器传送到模板，这样我们的信使就能说hello了。

在实现它之前，我们首先需要创建一个新路由。

### 一个新路由

我们将复用`HelloController`并添加一个新的`show`action。我们将在最后一条路由后添加一行，就像这样：

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser # Use the default browser stack.

  get "/", PageController, :index
  get "/hello", HelloController, :index
  get "/hello/:messenger", HelloController, :show
end
```

注意我们将原子`:messenger`放在了路径中。Phoenix会将URL中那个位置的值与键`messenger`进行[映射](http://elixir-lang.org/docs/stable/elixir/Map.html) ，并传送给控制器。

例如，如果我们打开[http://localhost:4000/hello/Frank](http://localhost:4000/hello/Frank) ，`:messenger`的值将会是"Frank"。

### 一个新action

对我们的新路由的请求将会由`HelloPhoenix.HelloController`的`show`action来处理。我们在`web/controllers/hello_controller.ex`已经有了那个控制器，所以我们要做的只是添加一个`show`action到其中。这一次，我们需要让映射中的一个参项被传送给action，这样我们就能把它（the messenger）传送给模板。为了做到这些，我们往控制器中添加了这个show函数：

```elixir
def show(conn, %{"messenger" => messenger}) do
  render conn, "show.html", messenger: messenger
end
```

这里有几件事需要注意。我们对传送给show函数的值做了模式匹配，这样`messenger`变量就会绑定到我们在URL中`:messenger`的位置上所填的值。比如，如果我们的URL是[http://localhost:4000/hello/Frank](http://localhost:4000/hello/Frank) ，messenger变量就会绑定到`Frank`。

在`show`action内部，我们也向render函数传递了第三个参数，一个键值对，键为`:messenger`，值为`messenger`变量。

> 注意： 如果action的主题需要访问一个包含了完整映射的参数变量，那么我们可以这样定义`show/2`：

```elixir
def show(conn, %{"messenger" => messenger} = params) do
  ...
end
```

这有助于记住`params`映射的键始终是字符串，等于号不代表赋值，而是一个[模式匹配](http://elixir-lang.org/getting-started/pattern-matching.html) 的断言。

### 一个新的模板

我们需要一个提供给`show`action的新页面。它将放在`web/templates/hello` 目录中，名为`show.html.eex`。它看上去很像我们的`index.html.eex`模板，除了需要显示messenger的名字。

为了实现它，我们将使用特殊的EEx标签来执行Elixir表达式 - `<%= %>`。注意到标签的起始部分有一个等号：`<%=`。它的意思是其中的任何Elixir代码都会被执行，且返回值会代替这个标签。如果等于号消失了，代码依旧会执行，但是值不会出现在页面。

模板看上去会是这样：

```html
<div class="jumbotron">
  <h2>Hello World, from <%= @messenger %>!</h2>
</div>
```

我们的messenger以`@messenger`形式出现。在这里，它不是一个模块属性。它是一个特殊的元编程的语法，代表`Dict.get(assigns, :messenger)`。它更易读且在模板中更容易使用。

我们完成了。如果你打开[http://localhost:4000/hello/Frank](http://localhost:4000/hello/Frank) ，你会看到一个这样的页面：

![Frank Greets Us from Phoenix](/images/hello-world-from-frank.png)

好好玩。无论你在`/hello/`后面输入什么，它都会作为你的messenger出现在页面上。
