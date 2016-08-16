首个教程的目的在于尽可能快地让一个Phoenix应用构建好并运行起来.

在我们开始前,请花一分钟阅读[Installation Guide](http://www.phoenixframework.org/docs/installation).安装好了所有必要的依赖,我们才能让应用流畅地构建和运行.

所以,我们需要安装Elixir, Erlang, Hex,和Phoenix.还需要安装好PostgreSQL和node.js用于构建一个默认应用.

Ok, 我们已经准备好了!

我们可以在任何目录中运行`mix phoenix.new`,来新建我们的Phoenix应用.Phoenix可接受目录的相对路径或绝对路径.假设我们的应用名为`hello_phoenix`,下列方法都可运作.

```console
$ mix phoenix.new /Users/me/work/elixir-stuff/hello_phoenix
```

```console
$ mix phoenix.new hello_phoenix
```

> 在开始前,我们要注意[Brunch.io](http://brunch.io/): Phoenix会默认使用Brunch.io进行资源管理.Brunch.io的依赖是通过node包管理工具安装的,而不是mix. Phoenix会在`mix phoenix.new`的末尾提示安装它们.如果我们选择`no`, 而之后没有通过`npm install`安装那些依赖,那么当我们试图启动应用时就会抛出错误，而且我们的资源可能没有合适地被载入。如果我们不想使用Brunch.io，我们可以简单地在`mix phoenix.new`之后加上`--no-brunch`。

现在，让我们运行`mix phoenix.new`，使用相对路径。


```console
mix phoenix.new hello_phoenix
* creating hello_phoenix/config/config.exs
* creating hello_phoenix/config/dev.exs
* creating hello_phoenix/config/prod.exs
...
* creating hello_phoenix/web/views/layout_view.ex
* creating hello_phoenix/web/views/page_view.ex

Fetch and install dependencies? [Yn]
```

Phoenix生成了目录结构，和我们的应用所需的所有文件。当完成后，它会询问我们是否需要安装依赖。让我们选择yes。

```console
Fetch and install dependencies? [Yn] Y
* running mix deps.get
* running npm install && node node_modules/brunch/bin/brunch build

We are all set! Run your Phoenix application:

    $ cd hello_phoenix
    $ mix phoenix.server

You can also run your app inside IEx (Interactive Elixir) as:

    $ iex -S mix phoenix.server

Before moving on, configure your database in config/dev.exs and run:

    $ mix ecto.create
```

当依赖安装好后，它会提示我们进入项目文件夹，并启动我们的应用。

Phoenix假设我们的PostgreSQL数据库中有一个`postgres`用户账号，它有正确的权限且密码是`postgres`。如果情况不符合，请查阅mix任务[ecto.create](http://www.phoenixframework.org/docs/mix-tasks#section--ecto-create-) 。

Ok，让我们来试一下。首先，我们会`cd`到刚创建的`hello_phoenix/`目录：

    $ cd hello_phoenix

现在，我们将创建我们的数据库：

```
$ mix ecto.create
The database for HelloPhoenix.Repo has been created.
```

注意：如果你是第一次运行这个命令，Phoenix可能会请你安装Rebar。请完成安装，Rebar是用于构建Erlang包的。

最后，我们将启动Phoenix服务器：

```console
$ mix phoenix.server
[info] Running HelloPhoenix.Endpoint with Cowboy using http on port 4000
23 Nov 05:25:14 - info: compiled 5 files into 2 files, copied 3 in 1724ms
```

如果我们选择了不让Phoenix在生成新应用时安装依赖，那么`phoenix.new`任务会在我们想要安装它们时，提示我们执行必要的步骤。

```console
Fetch and install dependencies? [Yn] n

We are all set! Run your Phoenix application:

    $ cd hello_phoenix
    $ mix deps.get
    $ mix phoenix.server

You can also run your app inside IEx (Interactive Elixir) as:

    $ iex -S mix phoenix.server

Before moving on, configure your database in config/dev.exs and run:

    $ mix ecto.create


Phoenix uses an optional assets build tool called brunch.io
that requires node.js and npm. Installation instructions for
node.js, which includes npm, can be found at http://nodejs.org.

After npm is installed, install your brunch dependencies by
running inside your app:

    $ npm install

If you don't want brunch.io, you can re-run this generator
with the --no-brunch option.
```

Phoenix默认在4000端口接收请求。如果我们在浏览器中输入[http://localhost:4000](http://localhost:4000) ，我们将看到Phoenix框架的欢迎页面。

![Phoenix Welcome Page](/images/welcome-to-phoenix.png)

如果你的屏幕看上去和图中一样，那么恭喜你！你现在有了一个正在运作的Phoenix应用。如果你看不到上面的页面，试着访问[http://127.0.0.1:4000](http://127.0.0.1:4000) 并确认你的系统将“localhost”设定为“127.0.0.1”。

本地，我们的应用运行于一个`iex`会话中。我们要按两次`ctrl-c`来停止它，就像停止普通的`iex`那样。

[下一步](http://www.phoenixframework.org/docs/adding-pages) 我们将稍稍自定义我们的应用，了解一下一个Phoenix应用是如何拼接起来的。
