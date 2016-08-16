路由器是Phoenix应用的中心。它们将匹配HTTP请求匹配到控制器动作，接通实时频道管理，并未作用域中间件定义了一系列的管道变换来设置路由。

Phoenix生成的router文件`web/router.ex`， 看上去会是这样：

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

你给定的应用名称将会替代router模块和控制器名称中的`HelloPhoenix`。

模块的第一行`use HelloPhoenix.Web, :router`，简单地使得Phoenix router函数在我们的router中可用。

Scopes在教程中有它自己的章节，所以我们不会花太多时间在`scope "/", HelloPhoenix do` 上。 `pipe_through :browser`这一行，在Pipeline章节会详细讲解。现在我们只需要知道，pipelines能为不同的routes执行一套中间件变换。

然而在scope块中，有我们的第一个route：

```elixir
  get "/", PageController, :index
```

`get`是一个Phoenix宏，它定义了`match/3`函数的一个从句。它对应了HTTP变量GET。类似的宏还对印着其它HTTP变量如POST，PUT，PATCH, DELETE, OPTIONS, CONNECT, TRACE 和 HEAD。

这些宏的第一个参数是路径。这里，是应用的根，`/`。后面的两个参数是我们希望用于处理这个请求的控制器和动作。这些宏还能接受别的设置，我们将在后面看到。

如果这是我们的router模块中唯一的route，那么宏展开后，`match/3`函数的从句会是：

```elixir
  def match(conn, "GET", ["/"])
```

`match/3`函数内部建立了连接，并调用了匹配到的控制其动作。

当我们添加更多route，更多match函数的从句会被添加到我们的router模块。它们会和Elixir中其它任何多从句函数一样运作。它们会被从头开始尝试，第一个和参项(verb和路径)匹配成功的从句会被执行。匹配成功后，搜索会停止，而且没有从句会再被尝试。

这意味着有可能基于HTTP verb和path创造一个永远不会被匹配的route，无论controller和action。

如果我们创造了一个有歧义的route，touter仍然回编译，但会发出警告。让我们来看看。

定义`scope "/", HelloPhoenix do`在router块的底部。

```elixir
get "/", RootController, :index
```

然后在你项目的根目录运行`$ mix compile`。你会看见以下警告：

```text
web/router.ex:1: warning: this clause cannot match because a previous clause at line 1 always matches
Compiled web/router.ex
```

Phoenix提供了一个伟大的工具来查看应用中的routes，mix任务`phoenix.routes`。

让我们看看它是如何工作的。进入Phoenix应用的根目录，运行`$ mix phoenix.routes`。（如果你没有完成上面的步骤，就需要运行`$ mix do deps.get, compile`，在运行`routes`任务之前。）你会看到类似下列的内容，由我们现在唯一的route生成：

```console
$ mix phoenix.routes
page_path  GET  /  HelloPhoenix.PageController :index
```

这个输出表明任何对应用根部的HTTP GET请求都会由`HelloPhoenix.PageController`中的`index`动作来操作。

`page_path`是一个Phoenix调用path helper的例子，我们很快会讲到它。

### 资源

除了类似`get`, `post`, 和 `put`的HTTP verbs，router还支持其它宏。其中最重要的是`resources`，它会展开成`match/3`的8个从句。

让我们在`web/router.ex`中添加一个resources：

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser # Use the default browser stack

  get "/", PageController, :index
  resources "/users", UserController
end
```

出于演示目的，就算没有`HelloPhoenix.UserController`也没关系。

然后进入项目根部，运行`$ mix phoenix.routes`

你会看到类似这样的东西：

```elixir
user_path  GET     /users           HelloPhoenix.UserController :index
user_path  GET     /users/:id/edit  HelloPhoenix.UserController :edit
user_path  GET     /users/new       HelloPhoenix.UserController :new
user_path  GET     /users/:id       HelloPhoenix.UserController :show
user_path  POST    /users           HelloPhoenix.UserController :create
user_path  PATCH   /users/:id       HelloPhoenix.UserController :update
           PUT     /users/:id       HelloPhoenix.UserController :update
user_path  DELETE  /users/:id       HelloPhoenix.UserController :delete
```

当然，你的项目名会取代`HelloPhoenix`。

这是一个HTTP verbs，paths，和控制器动作的标准矩阵。让我们分别查看它们，以稍稍不同的顺序。

- 对`/users`的GET请求会调用`index`动作来展示所有users。
- 对`/users/:id`的GET请求会调用`show`动作，附带id，来展示一个由ID选定的user
- 对`/users/new`的GET请求会调用`new`动作，来展示一个表格，用于创建新user。
- 对`/users`的POST请求会调用`create`动作，来保存一个新user到数据库。
- 对`/users/:id/edit`的GET请求会调用`edit`动作附带一个ID，以从数据库获取特定的user，并将信息呈现在一个表格中用于编辑。
- 对`/users/:id`的PATCH请求会调用`update`动作，附带一个ID，以保存更新后的user到数据库。
- 对`/users/:id`的PUT请求作用和上面一样。
- 对`/users/:id`的DELETE请求会调用`delete`动作，附带一个ID，以从数据库删除指定的user。

如果我们觉得用不到所有的routes，我们可以选择使用`:only`和`:except`选项。

假设我们有一个只读的posts resource。我们可以这样定义：

```elixir
resources "/posts", PostController, only: [:index, :show]
```

运行`$ mix phoenix.routes`，证明我们现在只有到index和show动作的routes定义了。


```elixir
post_path  GET     /posts      HelloPhoenix.PostController :index
post_path  GET     /posts/:id  HelloPhoenix.PostController :show
```

类似地，如果我们有一个comments resource，不希望提供删除的route，我们可以这样定义route。

```elixir
resources "/comments", CommentController, except: [:delete]
```

运行`$ mix phoenix.routes`，会发现我们除了对delete动作的DELETE请求没有，其它全都有。

```elixir
comment_path  GET     /comments           HelloPhoenix.CommentController :index
comment_path  GET     /comments/:id/edit  HelloPhoenix.CommentController :edit
comment_path  GET     /comments/new       HelloPhoenix.CommentController :new
comment_path  GET     /comments/:id       HelloPhoenix.CommentController :show
comment_path  POST    /comments           HelloPhoenix.CommentController :create
comment_path  PATCH   /comments/:id       HelloPhoenix.CommentController :update
              PUT     /comments/:id       HelloPhoenix.CommentController :update
```

### Path Helpers

Path helpers是为单个应用动态定义于`Router.Helpers`模块的函数。对我们来说，它是`HelloPhoenix.Router.Helpers`。它们的名字是从route定义所使用的控制器的名字中衍生出来的。我们的控制器是`HelloPhoenix.PageController`，而`page_path`是一个会返回到应用根部的path的函数。

百闻不如一见。在项目根部运行`$ iex -S mix`。当我们在router helpers上调用`page_path`函数，以`Endpoint`或connection和action作为参数，它会返回path。

```elixir
iex> HelloPhoenix.Router.Helpers.page_path(HelloPhoenix.Endpoint, :index)
"/"
```

这很有用，因为我们可以在模板中使用`page_path`函数来链接到我们的应用根部。注意：如果函数调用太长了，我们可以在应用的主view中包含`import HelloPhoenix.Router.Helpers`。

```html
<a href="<%= page_path(@conn, :index) %>">To the Welcome Page!</a>
```

更多信息，请移步[View Guide](http://www.phoenixframework.org/docs/views) 。

当我们必须修改router中route的path时，由于path helpers是基于routers动态定义的，所以模板中任何对`page_path`的调用都能正常工作。

### 关于Path Helpers的更多

当我们为user resource运行`phoenix.routes`任务时，它列出了path helper函数的输出，作为每一行的`user_path`。这里有每个action的翻译：

```elixir
iex> import HelloPhoenix.Router.Helpers
iex> alias HelloPhoenix.Endpoint
iex> user_path(Endpoint, :index)
"/users"

iex> user_path(Endpoint, :show, 17)
"/users/17"

iex> user_path(Endpoint, :new)
"/users/new"

iex> user_path(Endpoint, :create)
"/users"

iex> user_path(Endpoint, :edit, 37)
"/users/37/edit"

iex> user_path(Endpoint, :update, 37)
"/users/37"

iex> user_path(Endpoint, :delete, 17)
"/users/17"
```

如果path中有查询字符串呢？通过添加键值对作为可选参数，path helpers将会以查询字符串返回这些对。

```elixir
iex> user_path(Endpoint, :show, 17, admin: true, active: false)
"/users/17?admin=true&active=false"
```

如果我们需要一个完整url而非path呢？用`_url`代替`_path`就可以了：

```elixir
iex(3)> user_url(Endpoint, :index)
"http://localhost:4000/users"
```

应用端点马上会有它们自己的教程了。现在，就把它们当成会从router那里接管请求的实体。包括开启app/server，实施配置，以及实施所有请求共有的plugs。

`_url`函数会从每个环境的配置参数设置中获取宿主，端口，代理端口和SSL，这些构成完整URL所需的信息。我们将会在配置自己的章节详细地讨论它。现在，你可以看看`/config/dev.exs`文件中的值。

### 嵌套Resources

在Phoenix router中也可以嵌套resources。假设我们有一个`posts`资源，它和`users`是一对多的关系。意思是，一个人写好多文，而一篇文只属于一个人。我们可以通过在`web/router.ex`中添加一个嵌套route来表示它：

```elixir
resources "/users", UserController do
  resources "/posts", PostController
end
```

当我们运行`$ mix phoenix.routes`，我们得到了一些新的routes：

```elixir
. . .
user_post_path  GET     users/:user_id/posts           HelloPhoenix.PostController :index
user_post_path  GET     users/:user_id/posts/:id/edit  HelloPhoenix.PostController :edit
user_post_path  GET     users/:user_id/posts/new       HelloPhoenix.PostController :new
user_post_path  GET     users/:user_id/posts/:id       HelloPhoenix.PostController :show
user_post_path  POST    users/:user_id/posts           HelloPhoenix.PostController :create
user_post_path  PATCH   users/:user_id/posts/:id       HelloPhoenix.PostController :update
                PUT     users/:user_id/posts/:id       HelloPhoenix.PostController :update
user_post_path  DELETE  users/:user_id/posts/:id       HelloPhoenix.PostController :delete
```

我们看到这些routes都将posts限制到了一个user ID。我们会调用`PostController` `index`动作，但还会传送一个`user_id`。这暗示了我们会只显示某个user的所有posts。相同的限制应用于所有这些routes。

当我们为嵌套routes调用path helper函数时，我们需要按定义中的顺序传送IDs。例如下面的`show`route,`42`是`user_id`，`17`是`post_id`。记住在开始之前alias我们的`HelloPhoenix.Endpoint`。

```elixir
iex> alias HelloPhoenix.Endpoint
iex> HelloPhoenix.Router.Helpers.user_post_path(Endpoint, :show, 42, 17)
"/users/42/posts/17"
```

再一次，如果我们在函数调用的末尾添加一个键值对，它会添加到query字符串中。

```elixir
iex> HelloPhoenix.Router.Helpers.user_post_path(Endpoint, :index, 42, active: true)
"/users/42/posts?active=true"
```

### Scoped Routes

Scopes可以将有着相同path前缀的routes分组，并设置一套plug中间件。我们可能在管理员功能，APIs，尤其是版本化的APIs中会用到它。假设在一个站点，我们有用户生成检测，而进入这个检测需要管理员权限。这些资源的语义会很不同，而且它们可能不共用控制器。Scopes使我们能够隔离这些routes。

当users面对检测时，就像标准的资源一样。


```text
/reviews
/reviews/1234
/reviews/1234/edit
. . .
```

管理员检测路劲的前缀可以是`/admin`。

```text
/admin/reviews
/admin/reviews/1234
/admin/reviews/1234/edit
. . .
```

我们通过一个设置了path选项到`/admin`的scoped route来完成这些。现在，我们不要再嵌套这个scope到任何其它scope中了（就像`scope "/", HelloPhoenix do`）。


```elixir
scope "/admin" do
  pipe_through :browser

  resources "/reviews", HelloPhoenix.Admin.ReviewController
end
```

注意Phoenix会假设path都是以斜杠开头的，所以`scope "/admin" do` 和 `scope "admin" do`会产生相同结果。

还要注意，我们定义这个scope的方法，我们需要完整写出我们控制器的名称，`HelloPhoenix.Admin.ReviewController`。我们将在稍后修正它。

再次运行`$ mix phoenix.routes`，我们又得到了一些新的routes：

```elixir
. . .
review_path  GET     /admin/reviews           HelloPhoenix.Admin.ReviewController :index
review_path  GET     /admin/reviews/:id/edit  HelloPhoenix.Admin.ReviewController :edit
review_path  GET     /admin/reviews/new       HelloPhoenix.Admin.ReviewController :new
review_path  GET     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :show
review_path  POST    /admin/reviews           HelloPhoenix.Admin.ReviewController :create
review_path  PATCH   /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
             PUT     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
review_path  DELETE  /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :delete
```

看上去不错，但还有个问题。记得我们想要用户面对reviews routes`/reviews`，以及管理员的`/admin/reviews`。如果我们像这样引入用户面对的reviews到我们的router：

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser
  . . .
  resources "/reviews", ReviewController
  . . .
end

scope "/admin" do
  resources "/reviews", HelloPhoenix.Admin.ReviewController
end
```

然后运行`$ mix phoenix.routes`，我们会得到：

```elixir
. . .
review_path  GET     /reviews           HelloPhoenix.ReviewController :index
review_path  GET     /reviews/:id/edit  HelloPhoenix.ReviewController :edit
review_path  GET     /reviews/new       HelloPhoenix.ReviewController :new
review_path  GET     /reviews/:id       HelloPhoenix.ReviewController :show
review_path  POST    /reviews           HelloPhoenix.ReviewController :create
review_path  PATCH   /reviews/:id       HelloPhoenix.ReviewController :update
             PUT     /reviews/:id       HelloPhoenix.ReviewController :update
review_path  DELETE  /reviews/:id       HelloPhoenix.ReviewController :delete
. . .
review_path  GET     /admin/reviews           HelloPhoenix.Admin.ReviewController :index
review_path  GET     /admin/reviews/:id/edit  HelloPhoenix.Admin.ReviewController :edit
review_path  GET     /admin/reviews/new       HelloPhoenix.Admin.ReviewController :new
review_path  GET     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :show
review_path  POST    /admin/reviews           HelloPhoenix.Admin.ReviewController :create
review_path  PATCH   /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
             PUT     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
review_path  DELETE  /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :delete
```

看上去一切正常，除了每行开头的`review_path`。我们为用户面的和管理员的使用了同一个helper，这是不对的。我们可以通过添加一个`as: :admin`选项到我们的admin scope来修复它。

```elixir
scope "/", HelloPhoenix do
  pipe_through :browser
  . . .
  resources "/reviews", ReviewController
  . . .
end

scope "/admin", as: :admin do
  resources "/reviews", HelloPhoenix.Admin.ReviewController
end
```

现在`$ mix phoenix.routes`的结果就是我们想要的。

```elixir
. . .
      review_path  GET     /reviews           HelloPhoenix.ReviewController :index
      review_path  GET     /reviews/:id/edit  HelloPhoenix.ReviewController :edit
      review_path  GET     /reviews/new       HelloPhoenix.ReviewController :new
      review_path  GET     /reviews/:id       HelloPhoenix.ReviewController :show
      review_path  POST    /reviews           HelloPhoenix.ReviewController :create
      review_path  PATCH   /reviews/:id       HelloPhoenix.ReviewController :update
                   PUT     /reviews/:id       HelloPhoenix.ReviewController :update
      review_path  DELETE  /reviews/:id       HelloPhoenix.ReviewController :delete
. . .
admin_review_path  GET     /admin/reviews           HelloPhoenix.Admin.ReviewController :index
admin_review_path  GET     /admin/reviews/:id/edit  HelloPhoenix.Admin.ReviewController :edit
admin_review_path  GET     /admin/reviews/new       HelloPhoenix.Admin.ReviewController :new
admin_review_path  GET     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :show
admin_review_path  POST    /admin/reviews           HelloPhoenix.Admin.ReviewController :create
admin_review_path  PATCH   /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
                   PUT     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
admin_review_path  DELETE  /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :delete
```

path helper现在也会想我们预期的那样运作。运行`$ iex -S mix`并进行尝试。

```elixir
iex(1)> HelloPhoenix.Router.Helpers.review_path(Endpoint, :index)
"/reviews"

iex(2)> HelloPhoenix.Router.Helpers.admin_review_path(Endpoint, :show, 1234)
"/admin/reviews/1234"
```

如果我们有一些资源都需要管理员来处理呢？我们可以像这样把它们放到同一个scope中：

```elixir
scope "/admin", as: :admin do
  pipe_through :browser

  resources "/images",  HelloPhoenix.Admin.ImageController
  resources "/reviews", HelloPhoenix.Admin.ReviewController
  resources "/users",   HelloPhoenix.Admin.UserController
end
```

这里是`$ mix phoenix.routes`将会告诉我们的：

```elixir
. . .
 admin_image_path  GET     /admin/images            HelloPhoenix.Admin.ImageController :index
 admin_image_path  GET     /admin/images/:id/edit   HelloPhoenix.Admin.ImageController :edit
 admin_image_path  GET     /admin/images/new        HelloPhoenix.Admin.ImageController :new
 admin_image_path  GET     /admin/images/:id        HelloPhoenix.Admin.ImageController :show
 admin_image_path  POST    /admin/images            HelloPhoenix.Admin.ImageController :create
 admin_image_path  PATCH   /admin/images/:id        HelloPhoenix.Admin.ImageController :update
                   PUT     /admin/images/:id        HelloPhoenix.Admin.ImageController :update
 admin_image_path  DELETE  /admin/images/:id        HelloPhoenix.Admin.ImageController :delete
admin_review_path  GET     /admin/reviews           HelloPhoenix.Admin.ReviewController :index
admin_review_path  GET     /admin/reviews/:id/edit  HelloPhoenix.Admin.ReviewController :edit
admin_review_path  GET     /admin/reviews/new       HelloPhoenix.Admin.ReviewController :new
admin_review_path  GET     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :show
admin_review_path  POST    /admin/reviews           HelloPhoenix.Admin.ReviewController :create
admin_review_path  PATCH   /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
                   PUT     /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :update
admin_review_path  DELETE  /admin/reviews/:id       HelloPhoenix.Admin.ReviewController :delete
  admin_user_path  GET     /admin/users             HelloPhoenix.Admin.UserController :index
  admin_user_path  GET     /admin/users/:id/edit    HelloPhoenix.Admin.UserController :edit
  admin_user_path  GET     /admin/users/new         HelloPhoenix.Admin.UserController :new
  admin_user_path  GET     /admin/users/:id         HelloPhoenix.Admin.UserController :show
  admin_user_path  POST    /admin/users             HelloPhoenix.Admin.UserController :create
  admin_user_path  PATCH   /admin/users/:id         HelloPhoenix.Admin.UserController :update
                   PUT     /admin/users/:id         HelloPhoenix.Admin.UserController :update
  admin_user_path  DELETE  /admin/users/:id         HelloPhoenix.Admin.UserController :delete
```

很好，就是我们想要的，但我们可以让它变得更好。注意每个资源的控制器前都要加上`HelloPhoenix.Admin`。这很无聊也容易出错。我们可以添加一个`HelloPhoenix.Admin`选项到我们的scope声明中，就在scope path后面，这样所有的routes都会有正确完整的控制器名。

```elixir
scope "/admin", HelloPhoenix.Admin, as: :admin do
  pipe_through :browser

  resources "/images",  ImageController
  resources "/reviews", ReviewController
  resources "/users",   UserController
end
```

现在再次运行`$ mix phoenix.routes`，你会看到和之前一样的结果。

此外，我们可以将应用的所有routes嵌套进一个scope，它的别名就是我们的Phoenix应用的名字，这样能我们的控制器名称就不会重复。

在为新应用生成router时，Phoenix已经为我们这样做了（请看本节的开头）。注意在`defmodule`中对`HelloPhoenix.Router`的使用：

```elixir
defmodule HelloPhoenix.Router do
  use HelloPhoenix.Web, :router

  scope "/", HelloPhoenix do
    pipe_through :browser

    get "/images", ImageController, :index
    resources "/reviews", ReviewController
    resources "/users",   UserController
  end
end
```

再次运行`$ mix phoenix.routes`，结果表明我们所有的控制器都有正确的，完整的名字。

```elixir
image_path   GET     /images            HelloPhoenix.ImageController :index
review_path  GET     /reviews           HelloPhoenix.ReviewController :index
review_path  GET     /reviews/:id/edit  HelloPhoenix.ReviewController :edit
review_path  GET     /reviews/new       HelloPhoenix.ReviewController :new
review_path  GET     /reviews/:id       HelloPhoenix.ReviewController :show
review_path  POST    /reviews           HelloPhoenix.ReviewController :create
review_path  PATCH   /reviews/:id       HelloPhoenix.ReviewController :update
             PUT     /reviews/:id       HelloPhoenix.ReviewController :update
review_path  DELETE  /reviews/:id       HelloPhoenix.ReviewController :delete
  user_path  GET     /users             HelloPhoenix.UserController :index
  user_path  GET     /users/:id/edit    HelloPhoenix.UserController :edit
  user_path  GET     /users/new         HelloPhoenix.UserController :new
  user_path  GET     /users/:id         HelloPhoenix.UserController :show
  user_path  POST    /users             HelloPhoenix.UserController :create
  user_path  PATCH   /users/:id         HelloPhoenix.UserController :update
             PUT     /users/:id         HelloPhoenix.UserController :update
  user_path  DELETE  /users/:id         HelloPhoenix.UserController :delete
```

尽管scope像resource一样可以嵌套，但是不鼓励这种行为，因为这会使得我们的代码变得难以辨认。假设我们有一个版本控制的API，包含为images，reviews和users定义的resources。从技术上我们可以这样设置routes：


```elixir
scope "/api", HelloPhoenix.Api, as: :api do
  pipe_through :api

  scope "/v1", V1, as: :v1 do
    resources "/images",  ImageController
    resources "/reviews", ReviewController
    resources "/users",   UserController
  end
end
```

`$ mix phoenix.routes`表明我们得到了我们想要的routes。

```elixir
 api_v1_image_path  GET     /api/v1/images            HelloPhoenix.Api.V1.ImageController :index
 api_v1_image_path  GET     /api/v1/images/:id/edit   HelloPhoenix.Api.V1.ImageController :edit
 api_v1_image_path  GET     /api/v1/images/new        HelloPhoenix.Api.V1.ImageController :new
 api_v1_image_path  GET     /api/v1/images/:id        HelloPhoenix.Api.V1.ImageController :show
 api_v1_image_path  POST    /api/v1/images            HelloPhoenix.Api.V1.ImageController :create
 api_v1_image_path  PATCH   /api/v1/images/:id        HelloPhoenix.Api.V1.ImageController :update
                    PUT     /api/v1/images/:id        HelloPhoenix.Api.V1.ImageController :update
 api_v1_image_path  DELETE  /api/v1/images/:id        HelloPhoenix.Api.V1.ImageController :delete
api_v1_review_path  GET     /api/v1/reviews           HelloPhoenix.Api.V1.ReviewController :index
api_v1_review_path  GET     /api/v1/reviews/:id/edit  HelloPhoenix.Api.V1.ReviewController :edit
api_v1_review_path  GET     /api/v1/reviews/new       HelloPhoenix.Api.V1.ReviewController :new
api_v1_review_path  GET     /api/v1/reviews/:id       HelloPhoenix.Api.V1.ReviewController :show
api_v1_review_path  POST    /api/v1/reviews           HelloPhoenix.Api.V1.ReviewController :create
api_v1_review_path  PATCH   /api/v1/reviews/:id       HelloPhoenix.Api.V1.ReviewController :update
                    PUT     /api/v1/reviews/:id       HelloPhoenix.Api.V1.ReviewController :update
api_v1_review_path  DELETE  /api/v1/reviews/:id       HelloPhoenix.Api.V1.ReviewController :delete
  api_v1_user_path  GET     /api/v1/users             HelloPhoenix.Api.V1.UserController :index
  api_v1_user_path  GET     /api/v1/users/:id/edit    HelloPhoenix.Api.V1.UserController :edit
  api_v1_user_path  GET     /api/v1/users/new         HelloPhoenix.Api.V1.UserController :new
  api_v1_user_path  GET     /api/v1/users/:id         HelloPhoenix.Api.V1.UserController :show
  api_v1_user_path  POST    /api/v1/users             HelloPhoenix.Api.V1.UserController :create
  api_v1_user_path  PATCH   /api/v1/users/:id         HelloPhoenix.Api.V1.UserController :update
                    PUT     /api/v1/users/:id         HelloPhoenix.Api.V1.UserController :update
  api_v1_user_path  DELETE  /api/v1/users/:id         HelloPhoenix.Api.V1.UserController :delete
```

有趣的是，我们可以让多个scopes具有相同的path，只要我们注意不要重复routes。如果我们重复了route，将会得到类似的警告。

```console
warning: this clause cannot match because a previous clause at line 16 always matches
```

两个scopes定义了相同的path，这个router依旧一切正常。

```elixir
defmodule HelloPhoenix.Router do
  use Phoenix.Router
  . . .
  scope "/", HelloPhoenix do
    pipe_through :browser

    resources "/users", UserController
  end

  scope "/", AnotherApp do
    pipe_through :browser

    resources "/posts", PostController
  end
  . . .
end
```

`$ mix phoenix.routes`的结果。

```elixir
user_path  GET     /users           HelloPhoenix.UserController :index
user_path  GET     /users/:id/edit  HelloPhoenix.UserController :edit
user_path  GET     /users/new       HelloPhoenix.UserController :new
user_path  GET     /users/:id       HelloPhoenix.UserController :show
user_path  POST    /users           HelloPhoenix.UserController :create
user_path  PATCH   /users/:id       HelloPhoenix.UserController :update
           PUT     /users/:id       HelloPhoenix.UserController :update
user_path  DELETE  /users/:id       HelloPhoenix.UserController :delete
post_path  GET     /posts           AnotherApp.PostController :index
post_path  GET     /posts/:id/edit  AnotherApp.PostController :edit
post_path  GET     /posts/new       AnotherApp.PostController :new
post_path  GET     /posts/:id       AnotherApp.PostController :show
post_path  POST    /posts           AnotherApp.PostController :create
post_path  PATCH   /posts/:id       AnotherApp.PostController :update
           PUT     /posts/:id       AnotherApp.PostController :update
post_path  DELETE  /posts/:id       AnotherApp.PostController :delete
```

### Pipeline

距离第一次在router中看到`pipe_through :browser`已经过了很久了。现在让我们讲讲它。

记得在[Overview Guide](http://www.phoenixframework.org/docs/overview)中，我们曾吧plugs描述为被堆积并且能以一个预设的顺序执行，就像管道一样。现在我们将进一步了解这些plug堆栈是如何在router中工作的。

Pipelines就是plugs以特定的顺序堆在一起，然后起个名。它们允许我们自定义行为，并按照对请求的处理变形。Phoenix为我们提供了许多处理日常任务的pipelines。我们可以自定义它们，也可以创建新的pipeline，以适应我们的需求。

刚才生成的Phoenix应用定义了两个pipeline，叫做`:browser`和`:api`。在讨论它们之前，我们先谈谈Endpoint plugs中的plug堆。

##### The Endpoint Plugs

Endpoints组织了所有请求都要用到的plugs，并且应用了它们，在和它们基本的`:browser`,`:api`，以及自定义管道一起被派遣到router之前。默认的Endpoint plugs做了非常多工作。下面按顺序介绍它们。

- [Plug.Static](http://hexdocs.pm/plug/Plug.Static.html) - 服务静态资源。由于这个plug排在logger之前，所以静态资源是没有登记的。

- [Plug.Logger](http://hexdocs.pm/plug/Plug.Logger.html) - 记录来到的请求

- [Phoenix.CodeReloader](http://hexdocs.pm/phoenix/Phoenix.CodeReloader.html) - 为web目录中的所有路口开启了代码重载。它是在Phoenix应用中直接配置的。

- [Plug.Parsers](http://hexdocs.pm/plug/Plug.Parsers.html) - 解析请求的本体，当有一个解析器可用时。默认解析器能解析url编码，multipart和json（和poison一起）。当请求的内容类型不能被解析时，请求的本体就是无法触碰的。

- [Plug.MethodOverride](http://hexdocs.pm/plug/Plug.MethodOverride.html) - 为了POST请求，转换请求方法到PUT, PATCH 或 DELETE，伴随着一个合法的`_method`参数。

- [Plug.Head](http://hexdocs.pm/plug/Plug.Head.html) - 将HEAD请求转换为GET请求，并剥开响应体。

- [Plug.Session](http://hexdocs.pm/plug/Plug.Session.html) - 设置会话管理。
  注意`fetch_session/2`仍然必须在使用会话前准确地被调用，因为这个plug只是设置了如何获取会话。

- [Plug.Router](http://hexdocs.pm/plug/Plug.Router.html) - 将router放进请求的循环

##### `:browser` 和`:api`管道

Phoenix定义了另外两个默认的管道。`:browser`和`:api`。router会在匹配了一个route之后调用它们，假定我们已经在一个封闭的scope中使用它们调用了`pipe_through/1`。

就像它们的名字暗示的那样，`:browser`管道是为那些给浏览器渲染请求的routes提供的。`:api`管道视为那些给api产生数据的routes准备的。

`:browser`管道有五个plugs：`plug :accepts, ["html"]`定义了能被接受的请求格式，`:fetch_session`，会自然地获取会话并让其在连接中可用，`:fetch_flash`，会检索任何设置好的flash信息，以及`:protect_from_forgery`和 `:put_secure_browser_headers`,它们能保护posts免受跨站欺骗。

当前，`:api`管道只定义了`plug :accepts, ["json"]`。

router在route定义时调用管道，它们都在scope中。如果没有定义scope，router会在所有routes上调用管道。尽管不鼓励嵌套scope，但如果我们在一个嵌套scope中调用`pipe_through`，router将会先在父scope中导入`pipe_through`，然后是嵌套的scope。

多说无益，看看代码。

下面是我们的Phoenix应用的router，这次添加了一个api scope和一个route。

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
    pipe_through :browser

    get "/", PageController, :index
  end

  # Other scopes may use custom stacks.
  scope "/api", HelloPhoenix do
    pipe_through :api

    resources "/reviews", ReviewController
  end
end
```

当服务器收到一个请求，这个请求总是会先经过Endpoint上的plugs，然后它会试图匹配path和HTTP verb。

假设请求匹配了我们的第一个route：对`/`的GET。router会先将请求送至`:browser`管道 - 它会获取会话数据，获取flash，并执行欺骗保护 - 在请求被调遣到`PageController` `index` action之前。

相反，如果请求匹配到了宏`resources/2`定义的任何routes，router会将其送至`:api`管道 - 现在它什么都不会做 - 在将请求调遣到`HelloPhoenix.ReviewController`中正确的action之前。

如果我们知道我们的应用只会为浏览器渲染views，那么就可以通过删除`api`和scopes来简化router：


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

  pipe_through :browser

  get "/", HelloPhoenix.PageController, :index

  resources "/reviews", HelloPhoenix.ReviewController
end
```

删除了所有scopes意味着router必须对所有routes调用`:browser`管道。

让我们思考一下。如果我们需要将请求pipe不止一个管道呢？我们可以简单地`pipe_through`一个管道列表，Phoenix会按顺序调用它们。

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
  ...

  scope "/reviews" do
    # Use the default browser stack.
    pipe_through [:browser, :review_checks, :other_great_stuff]

    resources "/", HelloPhoenix.ReviewController
  end
end
```

这里有另一个例子，两个scopes有着不同的管道：

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
  ...

  scope "/", HelloPhoenix do
    pipe_through :browser

    resources "/posts", PostController
  end

  scope "/reviews", HelloPhoenix do
    pipe_through [:browser, :review_checks]

    resources "/", ReviewController
  end
end
```

通常，管道的scoping规则和你预期的一样。在这个例子中，所有routes会经过`:browser`管道。然后只有`reviews`资源routes会经过`:review_checks`管道。由于我们是以`pipe_through [:browser, :review_checks]`来声明管道的，所以Phoenix会按顺序调用列表中的每个管道。

##### 创造新管道

Phoenix允许我们在router的任何地方创建我们的自定义管道。我们只需要调用宏`pipeline/2`，伴随着这些参数：一个代表管道名的原子，一个包含了所需plugs的块。

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

  pipeline :review_checks do
    plug :ensure_authenticated_user
    plug :ensure_user_owns_review
  end

  scope "/reviews", HelloPhoenix do
    pipe_through :review_checks

    resources "/", ReviewController
  end
end
```

### 频道routes

频道是Phoenix中的实时组件，它很exciting。频道会为一个给定的主题处理进出socket的信息。频道routes，需要将请求匹配到socket和主题，才能调遣请求到正确的频道。（关于频道的更多信息，请看[Channel Guide](http://www.phoenixframework.org/docs/channels) ）。

我们为`lib/hello_phoenix/endpoint.ex`中的endpoint加装了socket处理器。Socket处理器会处理验证回调和频道routes。

```elixir
defmodule HelloPhoenix.Endpoint do
  use Phoenix.Endpoint

  socket "/socket", HelloPhoenix.UserSocket
  ...
end
```

然后，我们要打开`web/channels/user_socket.ex`文件，使用`channel/3`宏定义我们的频道routes。这些routes处理事件时会匹配一个频道的话题模式。如果我们有一个名为`RoomChannel`的频道模块，和一个叫做`"rooms:*"`的话题，那么代码可以这样写。


```elixir
defmodule HelloPhoenix.UserSocket do
  use Phoenix.Socket

  channel "rooms:*", HelloPhoenix.RoomChannel
  ...
end
```

话题只不过是字符串id。这种形式便于我们在一个字符串中定义话题和子话题 - “topic:subtopic”。`*`是一个通配符，它允许我们匹配任意子话题，所以`"rooms:lobby"`和`"rooms:kitchen"`都能匹配该route。

Phoenix抽象化了socket传送层，并使用两种传送机制 - WebSockets 和 Long-Polling。如果想让我们的频道只使用一种机制，我们可以设置`via`选项。

```elixir
channel "rooms:*", HelloPhoenix.RoomChannel, via: [Phoenix.Transports.WebSocket]
```

每个socket可以为多个频道处理请求。

```elixir
channel "rooms:*", HelloPhoenix.RoomChannel, via: [Phoenix.Transports.WebSocket]
channel "foods:*", HelloPhoenix.FoodChannel
```

我们可以把多个socket处理器堆放在endpoint中：

```elixir
socket "/socket", HelloPhoenix.UserSocket
socket "/admin-socket", HelloPhoenix.AdminSocket
```

### 总结

routing是个大话题，我们花了许多笔墨在这里。从本教程中你需要记住的是：
- 以HTTP verb名为开头的routes会展开成匹配函数的一个从句。
- 以'resources'为开头的routes会展开成匹配函数的八个从句。
- resources可以通过`only:`或`except:`选项调整匹配函数从句的数量。
- 任何routes都可以嵌套。
- 任何routes都可以被scoped到一个给定的path。
- 在scope中使用`as:`选项可以减少重复。
- 对scoped routes使用helper选项可以消除无法获取的paths。
