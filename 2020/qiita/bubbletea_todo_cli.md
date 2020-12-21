# はしがき

本記事ではElm Architecture likeにTUIアプリを作成できる[bubbletea](https://github.com/charmbracelet/bubbletea)というフレームワークを使ってToDoアプリを作っていきます。

タスク/完了したタスク一覧表示、追加、編集、完了、削除を行うことができます。
イメージ画像↓
![イメージ画像](https://i.gyazo.com/ffc1cc0b2ac62492cf33d945fe5775be.gif)

## Elm Architectureについて

Elm Architectureを知らない方は[公式ガイド(日本語訳)](https://guide.elm-lang.jp/architecture/)や[この記事](https://qiita.com/kazurego7/items/27a2b6f8b4a1bfac4bd3)をざっと読んでからにしたほうが理解がしやすいかと思います。(本記事ではElm Architectureの解説は殆どしません。)

筆者はElmを少し触ったことある程度なのでElm Architectureに対する理解が甘いです。
なので何か間違いがあればコメントや[Twitter](https://twitter.com/re_yuzuy)で指摘していただけると幸いです。

## 本記事で実装する機能

なお本記事ではタスク一覧表示と、タスクの追加までの解説(ガイド)にしようかと思っています。そこからあとはただただコード書き足してくだけなので。
適当にコードだけ読んで雰囲気を感じとりたい方、他の機能の実装の仕方を詳しく知りたい方、マサカリを投げてくださる方は[yuzuy/todo-cli](https://github.com/yuzuy/todo-cli)を参照してください！

それでは早速実装を始めていきましょう！

# 実装

## 筆者の環境
|   | バージョン |
|:-:|:-:|
| OS | macOS Catalina 10.15.7 |
| iTerm | 3.3.12 |
| Go | 1.15.2 |
| bubbletea | 0.7.0 |
| bubbles | 0.7.0 |

注: bubbleteaについて。このアプリを実装している途中にも破壊的変更が入ったので違うバージョンのbubbleteaを使う場合でバグが発生した場合は[リポジトリ](https://github.com/charmbracelet/bubbletea)を参照してください。

## タスクの一覧表示

この節ではタスク一覧を表示してカーソルでタスクを選択できるところまで実装します。

最初にパッケージを`go get`しましょう。

```
// bubbletea
go get github.com/charmbracelet/bubbletea

// utility
go get github.com/charmbracelet/bubbles
```

### Model

まずタスクの構造を定義します。

```go:main.go
type Task struct {
    ID        int
    Name      string
    IsDone    bool
    CreatedAt time.Time
}
```

ミニマムな構造になっているので他にも`FinishedAt`とか欲しい場合は追加しましょう。

次に`model`を定義します。Elm Architectureのモデルにあたるところです。
Elm Architectureではアプリの状態をモデルで管理します。
この章で
扱う状態はタスクの一覧とカーソルの位置の2つなので、実装は以下の通りになります。

```go:main.go
type model struct {
    cursor int
    tasks  []*Task
}
```

これだけではbubbleteaがモデルとして扱ってくれません。
モデルとして扱うには`model`に`tea.Model`を実装させる必要があります。

`tea.Model`の定義↓

```go
// Model contains the program's state as well as it's core functions.
type Model interface {
	// Init is the first function that will be called. It returns an optional
	// initial command. To not perform an initial command return nil.
	Init() Cmd

	// Update is called when a message is received. Use it to inspect messages
	// and, in response, update the model and/or send a command.
	Update(Msg) (Model, Cmd)

	// View renders the program's UI, which is just a string. The view is
	// rendered after every Update.
	View() string
}
```

まずは初期化関数である`Init()`から実装していきますが、このアプリでは最初に実行するべきコマンドがないので、nilを返すだけで大丈夫です。
(`model`structの初期化は別で行います。)
(本記事ではコマンドを殆ど扱わないのでコマンドについてはあまり気にしないでよいです。気になる場合は[Elm公式ガイド(日本語訳)](https://guide.elm-lang.jp/effects/)を参照してみるといいでしょう。)

```go:main.go
import (
    ...
    tea "github.com/charmbracelet/bubbletea"
)

...

func (m model) Init() tea.Cmd {
    return nil
}
```

### Update

`Update()`ではユーザーの操作(Msg)を元に`model`の状態を変化させます。

```go:main.go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "j":
            if m.cursor < len(m.tasks) {
                m.cursor++
            }
        case "k":
            if m.cursor > 1 {
                m.cursor--
            }
        case "q":
            return m, tea.Quit
        }
    }

    return m, nil
}
```

`tea.KeyMsg`(ユーザーのキー操作)をハンドリングして、
"j"が押されたらcursorの値を1増やす。
"k"が押されたらcursorの値を1減らす。
"q"が押されたら、`tea.Quit`コマンドを返してアプリを終了することを定義しています。

カーソルが無限に上がったり下がったりすると困るので`m.cursor > 1`などの条件を付けています。
ちなみに、`case "j", "down"`や`case "k", "up"`とすると、矢印キーでもカーソルを上下に動かすことができるようになるので、お好みでどうぞ。

### View

`View()`では`model`を元に描画するテキストを生成します。
`string`でゴリゴリ書いていきます。

```go:main.go
func (m model) View() string {
    s := "--YOUR TASKS--\n\n"

    for i, v := range m.tasks {
        cursor := " "
        if i == m.cursor-1 {
            cursor = ">"
        }

        timeLayout := "2006-01-02 15:04"
        s += fmt.Sprintf("%s #%d %s (%s)\n", cursor, v.ID, v.Name, v.CreatedAt.Format(timeLayout))
    }

    s += "\nPress 'q' to quit\n"

    return s
}
```

カーソルはインデックス番号で数えていないのでカーソルがそのタスクを指しているかは、インデックス`i`と`m.cursor-1`を比較して判定しています。

これでタスクを表示するのに十分な材料が揃いました！
`main`関数を定義してアプリを起動できるようにしましょう！

### main

`main`関数では`model`structの初期化、アプリの起動を行います。

```go:main.go
func main() {
    m := model{
        cursor: 1,
        tasks:  []*Task{
            {
                ID:        1,
                Name:      "First task!",
                CreatedAt: time.Now(),
            },
            {
                ID:        2,
                Name:      "Write an article about bubbletea",
                CreatedAt: time.Now(),
            },
        }
    }

    p := tea.NewProgram(m)
    if err := p.Start(); err != nil {
        fmt.Printf("app-name: %s", err.Error())
        os.Exit(1)
    }
}
```

本来ならタスクはファイル等から読み込みますが、そこまで実装すると時間がかかるので今回はハードコーディングします。
`tea.NewProgram()`でプログラムを生成、`p.Start()`で起動します。

早速`go run`コマンドで実行しましょう！
タスク一覧が表示され、"j","k"キーでカーソルを上下に移動させることができるはずです！

## タスクの追加

さて、タスクの一覧表示ができましたが、これではToDoアプリは名乗れそうにありません。
この節ではToDoアプリの最も大事な機能の一つ、タスクの追加を実装していきます。

### Model

先程まではカーソルの位置、タスク一覧を保持するだけで大丈夫でしたが、タスクの追加を実装するにあたってユーザーからの入力を受け取り、保持するフィールドを設ける必要があります。

bubbleteaでテキスト入力を実装するには`github.com/charmbracelet/bubbles/textinput`というパッケージを利用します。

```go:main.go
import (
    ...
    input "github.com/charmbracelet/bubbles/textinput"
)

type model struct {
    ...
    newTaskNameInput input.Model
}
```

これだけではタスク一覧表示モード(以下ノーマルモード)とタスク追加モード(以下追加モード)の判別ができないので、`mode`というフィールドも追加します。

```go:main.go
type model struct {
    mode int
    ...
    newTaskNameInput input.Model
}
```

`mode`の識別子も定義しましょう。

```go:main.go
const (
    normalMode = iota
    additionalMode
)
```

### Update

早速`Update()`関数の変更に入っていきたいところですが、タスクを追加するにあたって、足りない要素が1つ。
このToDoアプリではタスクのidを連番で管理しようと思っているので、最新のタスクのidを保持しておく必要があります。

グローバル変数として宣言、`main()`で初期化、`Update()`でインクリメント、という風に実装します。
(書いてる途中に思いましたが、`model`のフィールドとして管理するのもありかもしれません。)

```go:main.go
var latestTaskID int

func main() {
    ...
    // 今回はハードコーディング祭りですが、ちゃんと実装する場合はファイル等から読み込むときなどで初期値を入れましょう。
    latestTaskID = 2
    ...
}
```

さて、本題の`Update()`ですが、追加モードではノーマルモードとは違い全ての文字キーを入力として扱うことになります。

そのため`model.Update()`とは別に`model.addingTaskUpdate()`を定義して、`model.mode == additionalMode`であればそちらで処理するようにします。

ノーマルモードでは"a"が押されたときに追加モードに変更するように、
追加モードでは"ctrl+q"が押されたときにノーマルモードに戻るように、"enter"が押されたときにタスクが追加されるようにしましょう。

```go:main.go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    if m.mode == additionalMode {
        return m.addingTaskUpdate(msg)
    }

    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        ...
        case "a":
            m.mode = additionalMode
        ...
    }

    return m, nil
}

func (m model) addingTaskUpdate(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    case tea.KeyMsg:
        switch msg.String() {
        case "ctrl+q":
            m.mode = normalMode
            m.newTaskNameInput.Reset()
            return m, nil
        case "enter":
            m.tasks = append(m.tasks, &Task{
                ID  :      latestTaskID + 1,
                Name:      m.newTaskNameInput.Value(),
                CreatedAt: time.Now(),
            })

            latestTaskID++
            m.mode = normalMode
            m.newTaskNameInput.Reset()

            return m, nil
        }
    }

    var cmd tea.Cmd
    m.newTaskNameInput, cmd = input.Update(msg, m.newTaskNameInput)

    return m, cmd
}
```

`m.newTaskNameInput.Value()`でタイプされた値を取り出し、`m.newTaskNameInput.Reset()`でリセットしています。
`input.Update()`でキー入力をハンドリングして`m.newTaskNameInput`を更新しています。

### View

`View()`も`Update()`と同じく処理を分ける必要があります。
ですがこれはたった6行の変更で実装できます！

```go:main.go
func (m model) View() string {
    if m.mode == additionalMode {
        return m.addingTaskView()
    }
    ...
}

func (m model) addingTaskView() string {
    return fmt.Sprintf("Additional Mode\n\nInput a new task name\n\n%s", input.View(m.newTaskNameInput))
}
```

`input.View()`は`input.Model`を`View()`用に`string`へ変換してくれる変数です。

### main

新たなフィールド`mode`と`newTaskNameInput`の初期化を加えましょう。

```go:main.go
func main() {
    newTaskNameInput := input.NewModel()
    newTaskNameInput.Placeholder = "New task name..."
    newTaskNameInput.Focus()

    m := model{
        mode: normalMode,
        ...
        newTaskNameInput: newTaskNameInput,
    }
    ...
}
```

`Placeholder`は何も入力されていないときに表示する文字列です。

`Focus()`を呼び出すことによってその`input.Model`がフォーカスされます。
これは複数の入力を一画面で扱うときなんかに使うらしいです。今回は2つ以上の入力をさせるわけではないのでおまじない程度に思っていただければ大丈夫だと思います。

さあ、これでタスクの追加も実装できました！
早速先程同様、`go run`コマンド等で実行してみましょう！

# あとがき

本記事ではbubbleteaを用いたちょっとリッチなToDoアプリの作り方を解説しました。
思っていたより簡単に実装ができて、[tig](https://github.com/jonas/tig)のような複雑なCLIツールを作ってみたかったけどはじめの一歩を踏み出せていなかった僕にとってはとても魅力的なフレームワークでした。

bubbleteaにはこれ以外にもリッチなTUIアプリケーションを作るための機能がたくさんあるのでぜひ[リポジトリ](https://github.com/charmbracelet/bubbletea)を覗いてみてください！
