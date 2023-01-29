---
title: "discord.pyでわかったことまとめ"
slug: discord-py
author: ""
type: ""
date: 2023-01-29T07:39:36+09:00
draft: false
subtitle: ""
image: ""
toc: true
tocMove: true
tags: ["Discord", "discord.py", "Bot"]
---

これまで家族用に使っていた Slack の機能を徐々に Discord に移行している。  
[discord.py](https://discordpy.readthedocs.io/en/stable/) を使って Discord ボットを作る過程でわかったことをまとめる。

## テンプレートリポジトリ

discord.py のドキュメントを読んでもどのように実装しようか考えがまとまらず困っていたところ、以下のリポジトリを見つけた。

[kkrypt0nn/Python-Discord-Bot-Template: 🐍 A simple template to start to code your own and personalized discord bot in Python programming language.](https://github.com/kkrypt0nn/Python-Discord-Bot-Template)

Discord ボットの作成にはこのリポジトリを参考にした。

## わかったこと

Discord ボットは Python 3.10.9, discord.py 2.1.0 で実装している。  

### add_cog を使って動的にCogを読み込める

ドキュメントから Cog（コグ）の説明を引用する。

{{< blockquote author="discord.py" link="https://discordpy.readthedocs.io/ja/latest/ext/commands/cogs.html#ext-commands-cogs" >}}
Bot 開発においてコマンドやリスナー、いくつかの状態を 1 つのクラスにまとめてしまいたい場合があるでしょう。コグはそれを実現したものです。
{{< /blockquote >}}

コマンドを Cog クラスとして定義し、さらに Cog クラスごとにファイルを分けておくと管理が楽になる。  
しかし、 Cog を新たに定義するごとにボットへ追加するコードを書くのは面倒であるため、テンプレートリポジトリでは Cog を動的に読み込むコード[^1]が書いてある。

[^1]: [bot.py](https://github.com/kkrypt0nn/Python-Discord-Bot-Template/blob/main/bot.py)

動的読み込みのコードは割愛するが、読み込ませたいファイルには以下のように `setup` を書いておくだけでよい。

```python 
from discord.ext import commands

class ExampleCog(commands.Cog, name="example"):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

async def setup(bot):
    await bot.add_cog(ExampleCog(bot))
```

### Discordはスラッシュコマンドが便利

これは discord.py というより、Discord そのものについてだが。
スラッシュコマンド[^2]はコマンド入力中に候補が表示される、そのコマンドに必要な引数が表示される、など、入力がとても簡単である。  

[^2]: [Slash Commands FAQ – Discord](https://support.discord.com/hc/en-us/articles/1500000368501-Slash-Commands-FAQ)

`hybrid_command` デコレータを使うと、通常のコマンドとスラッシュコマンドのどちらでも使えるように定義できる。

```python
from discord.ext import commands

class ExampleCog(commands.Cog, name="example"):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.hybrid_command(name="testcommand", description="テストコマンド")
    async def test(self, ctx):
      await ctx.send("This is a hybrid command!")

async def setup(bot):
    await bot.add_cog(ExampleCog(bot))

```

### コマンドの引数の説明を追加できる

コマンドに `@discord.app_command.describe(msg="投稿するメッセージ")` というようなデコレータをつけることで、引数に説明文を追加できる。

他の方法として、 docstring に引数の説明を書くことでもデコレータと同様の説明を追加できることを確認している。

```python
from discord.ext import commands

class ExampleCog(commands.Cog, name="example"):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.hybrid_command(name="echo", description="入力したメッセージを投稿する")
    async def echo(self, ctx, msg: str):
        """
        エコーコマンド

        Parameters
        ---------------
        msg : str
            投稿するメッセージ
        """
        await ctx.send(f"{msg}")

async def setup(bot):
    await bot.add_cog(ExampleCog(bot))
```

### Discord上での引数名を変更できる

`rename` デコレータを使用することで、Discord 上で表示される引数名をコードとは別の値に変更できる。  
Discord 上では引数名を日本語で表示したい場合に便利。

```python
from discord import app_commands
from discord.ext import commands

class ExampleCog(commands.Cog, name="example"):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.hybrid_command(name="echo", description="入力したメッセージを投稿する")
    @app_commands.rename(msg="メッセージ")  # Discord上では msg ではなく、「メッセージ」という引数名となる
    async def echo(self, ctx, msg: str):
        """
        エコーコマンド

        Parameters
        ---------------
        msg : str
            投稿するメッセージ
        """
        await ctx.send(f"{msg}")

async def setup(bot):
    await bot.add_cog(ExampleCog(bot))
```

### ギルドコマンドとするにはデコレータの付与だけでは足りない

ギルドコマンドとは、特定のサーバでのみ有効なコマンドのこと。  
ギルドコマンドは即時に反映されるという特徴があるため、テンプレートリポジトリの README に `@app_commands.guilds()` デコレータを付与してギルドコマンドとする方法が書かれている。

しかし、私の環境では上記デコレータをつけるだけでは即時反映されなかった。  
デコレータ付与に加えて、 `bot.tree.sync(discord.Object(<GUILD ID>))` とすることで、即時反映されることを確認できた。
`bot.tree.sync は、` テンプレートリポジトリでは `bot.py` [^1] の `on_ready` 関数内に書かれている。

### 引数に型ヒントを書くとその型に変換してくれる

Slack では、 [slackbot](https://github.com/scrapinghub/slackbot) や [slack-machine](https://github.com/DonDebonair/slack-machine) を利用してボットを作っていたが、引数は文字列型なので数値を扱う場合には int や float へ変換する必要があった。

しかし、 discord.py では引数の型ヒントに記述した型に変換してくれる機能が備わっている。

```python
from discord.ext import commands

class ExampleCog(commands.Cog, name="example"):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @commands.hybrid_command(name="test", description="入力した数値に１を足して投稿する")
    async def test(self, ctx, number: int):
        await ctx.send(f"{number + 1}")

async def setup(bot):
    await bot.add_cog(ExampleCog(bot))
```

独自クラス (コンバーター) を作成して、より高度な変換も可能[^3]。  

[^3]: [コンバーター](https://discordpy.readthedocs.io/ja/latest/ext/commands/commands.html#converters)

### Cog での定期実行タスクは listener デコレータでスタートする

`commands.Cog.listener` デコレータをつけた `on_ready` メソッドで、 `tasks.loop` デコレータをつけたメソッドをスタートする。

```python
from discord.ext import commands, tasks

class ExampleCog(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot

    @tasks.loop(seconds=5.0)
    def example_task(self):
        print("example task")

    @commands.Cog.listener()
    async def on_ready(self):
        self.example_task.start()

async def setup(bot):
    await bot.add_cog(ExampleCog(bot))
```

以下のようにコンストラクタでスタートする方法も見受けられたが、自分の環境では動作しなかった。

```python
from discord.ext import commands, tasks

class ExampleCog(commands.Cog):
    def __init__(self, bot: commands.Bot):
        self.bot = bot
        self.example_task.start()

    @tasks.loop(seconds=5.0)
    def example_task(self):
        print("example task")
```

### 任意のチャンネルの取得

`bot.get_channel(<channel id>)` でチャンネルを取得できない現象に遭遇した。
正しいチャンネルの ID であっても None が返ってくる。

この現象について調べたところ、下記のブログを見つけた。

[【pycord】チャンネルが取得できない対処法【Python3.10】 - nekoy3`s room](https://nekoy3.net/2022/02/22/pycord-cant-get-channel-fix/)

上記ブログの通り、`bot.get_partial_messageable(<channel id>)` としたところ、チャンネルを取得できた。

