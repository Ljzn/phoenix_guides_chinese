Phoenix的controllers的作用像是中间模块。它们的函数 - 这里称为actions - 被从router中调用，来对HTTP请求做出回应。action会搜集所有必要的数据，完成所有必要的步骤，在调用view层去渲染模板或者返回JSON之前。

Phoenix controllers也是基于Plug包的，而且是它们自己的plugs。Controllers提供了几乎所有我们在action中会需要的东西。如果我们要寻找一些Phoenix controllers没有提供的东西，那么我们可能正在寻找Plug。请查看[Plug Guide](http://www.phoenixframework.org/docs/understanding-plug) 或 [Plug Documentation](http://hexdocs.pm/plug/) 。

我们刚生成的Phoenix应用中有一个`PageController`，它可以在`web/controllers/page_controller.ex`中找到。

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    render conn, "index.html"
  end
end
```

模块定义后的第一行，使用`use/1`宏调用了`HelloPhoenix.Web`模块，import了许多有用的函数。

`PageController`给了我们一个`index`动作，来呈现Phoenix欢迎页面。这个页面与在router中定义的默认route联系在一起。

### Actions

控制器动作就是函数。我们可以任意命名它们，只要符合Elixir的命名规则。唯一的要求是我们必须让action名能匹配router中的定义的一个route。

例如，在`web/router.ex`中，我们可以把默认的action名：

```elixir
get "/", HelloPhoenix.PageController, :index
```

修改成：

```elixir
get "/", HelloPhoenix.PageController, :test
```

只要把`PageController`中的动作名也改成`test`，欢迎界面就会照常显示。

```elixir
defmodule HelloPhoenix.PageController do
  . . .

  def test(conn, _params) do
    render conn, "index.html"
  end
end
```

尽管我们可以随意命名action，但为了方便，我们还是要遵守一些规则。我们在[Routing Guide](http://www.phoenixframework.org/docs/routing) 中讲过了，但再看一下。

- index   - 渲染一个给定资源类型的所有物品的列表
- show    - 渲染一个特定id的物品
- new     - 渲染一个表格用于创建新物品
- create  - 接受一个新物品的参项，并保存到数据库
- edit    - 通过ID检索一个物品，并将其参项显示到表格中以便编辑
- update  - 接受编辑过的物品的参项，并保存到数据库
- delete  - 接受需要删除的物品的ID，并从数据库中删除

每个action都有2个参项，由Phoenix提供。

第一个参项总是`conn`，一个包含了请求相关信息的结构，例如宿主，path elements，端口，query字符串，等等。`conn`是通过Elixir的Plug中间框架来实现的。跟多关于`conn`的信息请看[plug's documentation](http://hexdocs.pm/plug/Plug.Conn.html) 。

第二个参项是`params`。它是一个包含了HTTP请求中的任何参项的映射。经过与函数中的参项的模式匹配之后，它能生成渲染所需的数据。具体请看[Adding Pages guide](http://www.phoenixframework.org/docs/adding-pages)，我们添加一个messenger参项到`web/controllers/hello_controller.ex`中的`show`route。


```elixir
defmodule HelloPhoenix.HelloController do
  . . .

  def show(conn, %{"messenger" => messenger}) do
    render conn, "show.html", messenger: messenger
  end
end
```

某些情况下 - 通常是在`index`动作中 - 我们不在乎参项，因为我们的行为不依赖于它们。此时，我们不使用收到的params，并且在变量名前加上下划线，`_params`。这会让编译器不发出“有未使用变量”的警告，同时也保持正确的参数个数。

### 收集数据

尽管Phoenix没有装载它自己的数据获取层，Elixir项目[Ecto](http://hexdocs.pm/ecto) 提供了一个非常好的解决方案，对于使用[Postgres](http://www.postgresql.org/) 关系数据库。我们在[Ecto Models Guide](http://www.phoenixframework.org/docs/ecto-models) 中会讲解如何在Phoenix项目中使用Ecto。Ecto所支持的数据库[Usage section of the Ecto README](https://github.com/elixir-lang/ecto#usage) 。

当然，这里还有许多其它的数据获取方法。[Ets](http://www.erlang.org/doc/man/ets.html) 和 [Dets](http://www.erlang.org/doc/man/dets.html)键值数据存储，组成了[OTP](http://www.erlang.org/doc/) 。OTP也提供了一个关系数据库，叫做[mnesia](http://www.erlang.org/doc/man/mnesia.html) ，用的是它自己的query语言，叫做QLC。Elixir和Erlang都有许多库，可以用于操作各种流行的数据存储。

数据对你很重要，但在本教程中没有涵盖这些设置。

### Flash 信息

有时我们需要在动作执行后与用户交流。也许是在更新一个model时出现了一个错误。也许我们只想欢迎用户回来。所以，我们有了Flash信息。

`Phoenix.Controller`模块提供了`put_flash/3`和 `get_flash/2`函数，来帮助我们设置和检索flash信息，以键值对的形式。让我们在`HelloPhoenix.PageController`中设置两个flash信息。

将`index`动作修改为：

```elixir
defmodule HelloPhoenix.PageController do
  . . .
  def index(conn, _params) do
    conn
    |> put_flash(:info, "Welcome to Phoenix, from flash info!")
    |> put_flash(:error, "Let's pretend we have an error.")
    |> render("index.html")
  end
end
```

`Phoenix.Controller`模块并不限制我们使用的键。只要内部一致，什么键都可以。`:info`和`:error`只是常用的罢了。

为了看见我们的flash信息，我们需要检索它们，并在template/layout中显示它们。完成检索的一种方式就是使用`get_flash/2`，它的参数是`conn`和我们的键。然后它就会返回那个键的值。

幸运的是，我们的应用layout，`web/templates/layout/app.html.eex`，已经把这些弄好了。


```html
<p class="alert alert-info" role="alert"><%= get_flash(@conn, :info) %></p>
<p class="alert alert-danger" role="alert"><%= get_flash(@conn, :error) %></p>
```

重新载入[Welcome Page](http://localhost:4000/)，就能看到它们了。

除了`put_flash/3`和`get_flash/2`,  `Phoenix.Controller`中还有一个值得了解的函数。`clear_flash/1`函数，参数是`conn`,会删除会话中的所有flash信息。

### 渲染

控制器有许多种方法来渲染内容。最简单的就是渲染一些文本，使用Phoenix提供的`text/2`函数。

假设我们有一个`show` action，它从params映射汇总接收id，然后我们要做的就是返回这个id以及一些文本。我们能这样做。

```elixir
def show(conn, %{"id" => id}) do
  text conn, "Showing id #{id}"
end
```

假设我们已经有一个route，`get "/our_path/:id"`，指向这个`show` action，在浏览器中访问`/our_path/15`将会显示`Showing id 15`这段不带任何HTML格式的文本。

另一种是使用`json/2`函数渲染纯JSON。我们需要传送给它一些可以由[Poison library](https://github.com/devinus/poison) 解析成JSON的东西，例如一个映射。（Poison是Phoenix的依赖之一）

```elixir
def show(conn, %{"id" => id}) do
  json conn, %{id: id}
end
```

如果我们再次访问`our_path/15`，我们会看到一个JSON块，内容是键`id`指向数字`15`。

```json
{"id": "15"}
```

Phoenix控制器也可以不用模板渲染HTML。没错，就是`html/2`函数。

```elixir
def show(conn, %{"id" => id}) do
  html conn, """
     <html>
       <head>
          <title>Passing an Id</title>
       </head>
       <body>
         <p>You sent in id #{id}</p>
       </body>
     </html>
    """
end
```

访问`/our_path/15`，现在渲染的是在`show`中定义的HTML字符串，其中插值了`15`。注意我们没有用`eex`模板。这是一个多行字符串，所以我们的插值用的是`#{id}`而不是`<%= id %>`。

`text/2`, `json/2`, 和 `html/2`函数，不需要渲染Phoenix view，或者模板。但这不是重点。

`json/2`函数很明显适用于编写API，其它两者可能也很好用，但是渲染一个带值得模板到layout中，是很常见的。

所以，Phoenix提供了`render/3`函数。

在内部，`render/3`定义与`Phoenix.View`模块，而不是`Phoenix.Controller`，但为了方便，它在`Phoenix.Controller`中有别名。

在[Adding Pages Guide](http://www.phoenixframework.org/docs/adding-pages) 中，我们已经看过了render函数。在`web/controllers/hello_controller.ex`中，我们的`show`action会是这样。

```elixir
defmodule HelloPhoenix.HelloController do
  use HelloPhoenix.Web, :controller

  def show(conn, %{"messenger" => messenger}) do
    render conn, "show.html", messenger: messenger
  end
end
```

为了使`render/3`正常工作，控制器的名称和对应view的名要相同，而view名要和`show.html.eex`模板所在目录的名称相同。换句话说，`HelloController` 需要`HelloView`, 而 `HelloView` 需要 `web/templates/hello` 目录, 其中必须包含 `show.html.eex` 模板。

`render/3`也传送了`show`动作从params散列中为`messenger`接收到的值给模板，用于插值。

如果在使用`render`时，我们需要传送值给模板，这很简单。我们可以像刚才那样传送一个词典，`messenger: messenger`，或者使用`Plug.Conn.assign/3`，它能方便地返回`conn`。

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> render("index.html")
end
```

注意： `Phoenix.Controller`imports了`Plug.Conn`，所以用短语，`assign/3`，是没问题的。

我们可以在`index.html.eex`模板中获取这个信息，或者是在我们的layout中，使用`<%= @message %>`。

传送更多的值到模板中，就和把`assign/3`函数堆积在管道中一样简单。

```elixir
def index(conn, _params) do
  conn
  |> assign(:message, "Welcome Back!")
  |> assign(:name, "Dweezil")
  |> render("index.html")
end
```

这样，在`index.html.eex`模板中，`@message` 和`@name` 都是可用的。

如果我们想有一个默认的欢迎信息，而且可由一些action重写，要怎么做？很简单，只需要在`conn`通往控制器action的路上，使用`plug`来变换它。

```elixir
plug :assign_welcome_message, "Welcome Back"

def index(conn, _params) do
  conn
  |> assign(:name, "Dweezil")
  |> render("index.html")
end

defp assign_welcome_message(conn, msg) do
  assign(conn, :message, msg)
end
```

如果我们想让 plug `assign_welcome_message`只适用于一部分actions呢？Phoenix提供了一个方法，使得可以选择那些action会使用一个plug。如果我们希望`plug :assign_welcome_message`只作用于`index`和`show`actions，我们可以这样做。

```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  plug :assign_welcome_message, "Hi!" when action in [:index, :show]
. . .
```

### 直接发送响应

如果上述的渲染方法都不符合我们的需求，我们可以自己使用plug提供的函数来定制。假设我们想发送一个响应，Status是“201”。我们可以用`send_resp/3`函数来实现。

```elixir
def index(conn, _params) do
  conn
  |> send_resp(201, "")
end
```

重新载入[http://localhost:4000](http://localhost:4000)将会呈现一个完全的空白页面。但在我们的浏览器开发者工具的network一栏中，可以看到接收了一个Status为201的响应。

如果我们想指定内容的类型，我们可以用，`put_resp_content_type/2`，`send_resp/3`的组合。

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/plain")
  |> send_resp(201, "")
end
```

以这种方式使用Plug函数，我们可以打造自己需要的响应。

尽管渲染不易模板为结尾。但默认的，模板渲染的结果会插入到layout中，而layout也会被渲染。

[Templates and layouts](http://www.phoenixframework.org/docs/templates) 有它们自己的教程，所以这里就不多讲了。我们将关注的是如何在控制器动作内，指定不同的layout，或都不用。

### 指定layouts

Layouts只不过是一种特殊的模板。它们存在于`/web/templates/layout` 中。Phoenix在我们生成app时为我们创建了一个。它叫做`app.html.eex`，所有模板都会默认在它之中渲染。

因为layouts只是模板，所以它们需要一个view来渲染。那就是定义在`/web/views/layout_view.ex`中的`LayoutView`模块。所以我们只需要把想要渲染的layouts放到`/web/templates/layout`目录中就可以了。

在创建新layout之前，让我们不用layout渲染一个模板。

`Phoenix.Controller`模块提供了`put_layout/2`函数，用于选择layouts。它的第一个参数是`conn`以及我们想要渲染的layout的名字的字符串。函数另一个从句会为第二个参数匹配布尔值`false`，这也是我们如何不带layout地渲染Phoenix欢迎页面。

在一个新生成的Phoenix app中，编辑`PageController` 模块`web/controllers/page_controller.ex`中的`index`action。

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```

重载[http://localhost:4000/](http://localhost:4000/) ，我们会看到一个非常不同的页面，没有标题，图片，或是css风格。

有一点很重要！对于这种在管道中间被调用的函数，比如这里的`put_layout/2`，需要用括号包裹参数，因为管道操作结合得非常紧密。否则很可能导致解析错误，和非常奇怪的结果。

如果你得到了像这样的stack trace，

```text
**(FunctionClauseError) no function clause matching in Plug.Conn.get_resp_header/2

Stacktrace

    (plug) lib/plug/conn.ex:353: Plug.Conn.get_resp_header(false, "content-type")
```

在这里你的参数取代了`conn`成为了第一个参数，首先要检查的就是括号有没写好。

这是对的：

```elixir
def index(conn, params) do
  conn
  |> put_layout(false)
  |> render "index.html"
end
```

这是错的：

```elixir
def index(conn, params) do
  conn
  |> put_layout false
  |> render "index.html"
end
```

现在让我们创造另一个layout，并往其中渲染一个index模板。作为例子，我们的这个layout是提供给admin的，在其中没有logo图片。为了做到它，让我们复制已存的`app.html.eex`，到同一个目录`web/templates/layout`下的新文件`admin.html.eex`。然后让我们删除`admin.html.eex`中显示logo的那一行。

```html
<span class="logo"></span> <!-- remove this line -->
```

然后，将新layout的basename传送给`web/controllers/page_controller.ex`中的`index` action里的`put_layout/2`。

```elixir
def index(conn, params) do
  conn
  |> put_layout("admin.html")
  |> render "index.html"
end
```

当载入页面时，我们会渲染admin layout，不带logo。

### 重写渲染格式

模板渲染十分好，能否随意改格式？假设我们有时需要文本，有时需要JSON，有时需要HTML。那要怎么办？

Phoenix允许我们在fly时通过`_format` query string参项来修改格式。为了实现它，Phoenix要求一个合适命名的view，和一个合适命名的模板，在正确的目录中。

作为例子，让我们看看刚生成的应用的`PageController`中的 index action。这里有正确的view，`PageView`，正确的模板目录，`/web/templates/page`，和正确的模板用于渲染HTML，`index.html.eex`。

```elixir
def index(conn, _params) do
  render conn, "index.html"
end
```

缺少的是一个可供替代的模板，用于渲染文本。让我们添加一个`/web/templates/page/index.text.eex`。这是我们的`index.text.eex`模板。

```elixir
OMG, this is actually some text.
```

还有一点点工作要做。我们需要告诉router，应该接受`text`格式。需要往`:browser` 的管道中的接受格式列表里，添加`text`。让我们打开`web/router.ex`，并修改`plug :accepts`，让其包含`text`和`html`。

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  pipeline :browser do
    plug :accepts, ["html", "text"]
    plug :fetch_session
    plug :protect_from_forgery
    plug :put_secure_browser_headers
  end
. . .
```

我们还需要告诉控制器，在渲染模板时使用和`Phoenix.Controller.get_format/1`的返回值相同的模板。只需将模板`"index.html"`替换为`:index`。

```elixir
def index(conn, _params) do
  render conn, :index
end
```

如果我们打开[http://localhost:4000/?_format=text](http://localhost:4000/?_format=text), 会看到 `OMG, this is actually some text.`

当然，我们也可以传送数据到模板。让我们删除函数定义中`params`前面的`_`，这样我们的action就会接受一个信息作为参项。这一次，我们会使用较少弹性的字符串版本的text模板名，只是为了看看它能否工作。

```elixir
def index(conn, params) do
  render conn, "index.text", message: params["message"]
end
```

让我们丰富一下text模板。

```elixir
OMG, this is actually some text. <%= @message %>
```

现在，打开`http://localhost:4000/?_format=text&message=CrazyTown`，我们会看到"OMG, this is actually some text. CrazyTown"

### 设置内容类型

和`_format` query string param相似，我们可以通过修改HTTP内容类型头文件并提供合适的模板，来渲染任何格式。

如果我们想要渲染xml版本的`index` action，我们需要在`web/page_controller.ex`里这样写。

```elixir
def index(conn, _params) do
  conn
  |> put_resp_content_type("text/xml")
  |> render "index.xml", content: some_xml_content
end
```

我们还需要提供一个`index.xml.eex`模板，其中是合法的xml，然后就完成了。

想得到合法的mime类型的列表，请查看[mime.types](https://github.com/elixir-lang/mime/blob/master/lib/mime.types) 文档。

### 设置HTTP Status

和设置内容类型的方法类似，我们也可以设置一个相应的HTTP状态码。`Plug.Conn`模块，已经被import到了所有控制器，具有一个`put_status/2`函数来实现。

`put_status/2`第一个参数是`conn`，第二个参数要么是个整数，要么是一个我们想要设置成状态码的"friendly name"，作为原子来使用。这里是所支持的[friendly names](https://github.com/elixir-lang/plug/blob/master/lib/plug/conn/status.ex#L7-L66) 的列表。

让我们修改`PageController` `index` action中的状态。

```elixir
def index(conn, _params) do
  conn
  |> put_status(202)
  |> render("index.html")
end
```

我们给定的状态码必须合法，否则 [Cowboy](https://github.com/ninenines/cowboy) ，Phoenix运行于的web服务器，就会抛出错误。如果我们查看开发日志（这里指iex会话），或者使用浏览器的网络检查工具，我们将会在重载页面时看见设置好的状态码。

如果action发送一个响应 - 渲染或者重定向 - 改变代码将不会改变响应的行为。例如，如果我们将status设置成404或500，然后`render "index.html"`，我们将不会得到错误页面。同样，没有一个300状态码会真的重定向。（它不知道重定向到哪里，即使代码确实影响了行为。）

下列`HelloPhoenix.PageController` `index` action的代码，将不会像预期的那样渲染默认的`not_found`行为。

```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render("index.html")
end
```

从`HelloPhoenix.PageController`中渲染404页面的正确方法是：

```elixir
def index(conn, _params) do
  conn
  |> put_status(:not_found)
  |> render(HelloPhoenix.ErrorView, "404.html")
end
```

### 重定向

通常，在一个请求的中间，我们需要重定向到一个新的url。一个成功的`create`动作，在实际中，通常会重定向到`show`动作，为我们刚创建的model。也可以重定向到`index`动作，来展示同类的所有东西。还有许多情形下，重定向也很有用。

不管什么情况，Phoenix控制器提供的`redirect/2`函数都能让重定向变简单。Phoenix区别对待应用内的重定向，和重定向到url - 应用内的或外部的。

为了测试`redirect/2`，让我们在`web/router.ex`中创造一个新的route。


```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router
  . . .

  scope "/", HelloPhoenix do
    . . .
    get "/", PageController, :index
  end

  # New route for redirects
  scope "/", HelloPhoenix do
    get "/redirect_test", PageController, :redirect_test, as: :redirect_test
  end
  . . .
end
```

然后我们会让`index` action重定向到我们的新route。

```elixir
def index(conn, _params) do
  redirect conn, to: "/redirect_test"
end
```

最后，让我们在同一个文件中定义我们重定义到的action，它简单地渲染了文本`Redirect!`。

```elixir
def redirect_test(conn, _params) do
  text conn, "Redirect!"
end
```

当我们重载[Welcome Page](http://localhost:4000)，会看到我们已经重定向到了`/redirect_test`，它渲染了文本`Redirect!`。成功了！

我们可以打开开发者工具，点击网络那一栏，再次访问我们的根route。我们看到该页面有两个主要请求 - 一个对`/`的get，附带`302`状态，以及一个对`/redirect_test`的get，附带`200`状态。

注意到重定向函数的参数是`conn`和一个代表应用内相关路径的字符串。它的参数也可以是`conn`和一个代表完整url的字符串。

```elixir
def index(conn, _params) do
  redirect conn, external: "http://elixir-lang.org/"
end
```

我们也可以使用在[Routing Guide](http://www.phoenixframework.org/docs/routing)中学过的path helpers。`alias`那些`web/router.ex`中的helpers，可以缩短表达式。


```elixir
defmodule HelloPhoenix.PageController do
  use HelloPhoenix.Web, :controller

  def index(conn, _params) do
    redirect conn, to: redirect_test_path(conn, :redirect_test)
  end
end
```

注意下面的是错的。因为`:to`不接受path之外的东西。

```elixir
def index(conn, _params) do
  redirect conn, to: redirect_test_url(conn, :redirect_test)
end
```

如果我们想使用url helper来传送完整的url到`redirect/2`，我们必须使用原子`:external`。注意url不一定要是外部的。

```elixir
def index(conn, _params) do
  redirect conn, external: redirect_test_url(conn, :redirect_test)
end
```
