---
title: "My new name is... Rain Man"
date: 2025-02-13
description: "Building a Matrix rain effect in the terminal using Go and Bubble Tea"
repository: "https://github.com/isopath/matrix.go"
---

![Matrix Rain Demo](/matrix.gif)

We got bored one weekend and decided to build a Matrix rain effect in the terminal. You know, those cascading green characters from the movie? But we wanted ours to display actual text files, not just random symbols. Here's how we built it.

## The Problem with Shipping Assets

Building a terminal app is easy until you need to include data files. We had a bunch of text files we wanted to display, but asking users to download separate assets felt wrong. What if they put them in the wrong folder? What if they forgot entirely?

Go's `//go:embed` directive solves this beautifully. It lets you embed files directly into the compiled binary. No external dependencies, no "file not found" errors.

```go
//go:embed assets/*.txt
var assetsFS embed.FS

func readAsset(filename string) (string, error) {
    data, err := assetsFS.ReadFile(filename)
    if err != nil {
        return "", err
    }
    return string(data), nil
}
```

Now our single binary contains everything. Ship it, run it, done.

## Building a Menu That Doesn't Suck

We wanted users to pick which text file to rain-ify. Bubble Tea (Charm's TUI framework) makes this possible, but it's not exactly plug-and-play. The API requires you to jump through some hoops.

First, every list item needs to implement the `list.Item` interface. Go is strict about this - even if you don't use a feature, you still need to satisfy the interface:

```go
type item struct {
    title    string
    filename string
}

// Required by list.Item, even though we disable filtering
func (i item) FilterValue() string { 
    return "" 
}
```

Then there's the delegate - the thing that actually renders each item. This is where you customize the look:

```go
type itemDelegate struct{}

func (d itemDelegate) Render(w io.Writer, m list.Model, index int, listItem list.Item) {
    i, ok := listItem.(item)
    if !ok {
        return  // Trust but verify
    }

    str := fmt.Sprintf("%d. %s", index+1, i.title)
    
    if index == m.Index() {
        fmt.Fprint(w, selectedItemStyle.Render("> "+str))
    } else {
        fmt.Fprint(w, itemStyle.Render(str))
    }
}
```

The type assertion `i, ok := listItem.(item)` is Go's way of saying "I know what this should be, but prove it." It's defensive programming that prevents panics when types don't match.

## The Heartbeat of Animation

Bubble Tea uses the Elm Architecture: your app has a Model (state), an Update function (handles messages), and a View function (renders output). Everything flows through messages.

For animation, we needed a timer. Enter `tea.Tick`:

```go
func tick() tea.Cmd {
    return tea.Tick(time.Millisecond*80, func(t time.Time) tea.Msg {
        return tickMsg(t)
    })
}
```

Every 80 milliseconds, this fires a message. Why 80ms? We tried 16ms (60fps) first, but the laptop fan sounded like a jet engine. 80ms is smooth enough to look good without melting your CPU.

The `Update` function catches these ticks and advances the animation:

```go
func (m model) Update(msg tea.Msg) (tea.Model, tea.Cmd) {
    switch msg := msg.(type) {
    
    case tickMsg:
        if m.viewing {
            m.updateColumns()  // Move the rain
            return m, tick()   // Schedule next frame
        }
    }
    // ... handle other messages
}
```

Notice how `tick()` returns itself? That's the recursive heartbeat. Each frame schedules the next one. Elegant once you wrap your head around it.

## Making It Rain

The actual rain effect works by creating vertical columns of characters. Each column has a position, a height, and an offset into the source text:

```go
type column struct {
    x      int // horizontal position  
    height int // how many characters
    offset int // position in source text
}
```

We calculate how many columns fit on screen (one every 2 characters for spacing), then randomize their starting positions:

```go
numCols := m.width / 2
m.columns = make([]column, numCols)

for i := range m.columns {
    m.columns[i] = column{
        x:      i * 2,
        height: rand.Intn(m.height) + 5,
        offset: rand.Intn(len(runes)),
    }
}
```

Each frame, every column increments its offset. When it reaches the end of the text, it wraps back to zero. That's what creates the infinite scrolling effect. We also randomize heights occasionally - real rain isn't uniform, and neither should this be.

## The View from Above

Rendering is the fun part. We build a 2D grid representing the terminal screen, fill it with spaces, then paint our rain columns:

```go
grid := make([][]rune, m.height)
for i := range grid {
    grid[i] = make([]rune, m.width)
    for j := range grid[i] {
        grid[i][j] = ' '
    }
}
```

Then we iterate through columns and place characters. The math `(col.offset + row) % len(runes)` wraps the text index so columns seamlessly loop through the content.

Finally, we convert this grid to a string with colors. Using `strings.Builder` instead of string concatenation keeps memory allocations sane - important when you're doing this 12 times per second.

## Lessons Learned

**Embedding is underrated.** Shipping a single binary with all assets included is pure joy. No installers, no PATH issues, no "it works on my machine."

**Unicode will bite you.** We used `[]rune` everywhere instead of `string` indexing. Runes properly handle multi-byte characters (emojis, CJK scripts, etc.). String indexing would break spectacularly on anything beyond ASCII.

**Bubble Tea has a learning curve.** The Elm Architecture feels foreign at first - every update returns a new state and commands. But once it clicks, building TUIs becomes genuinely fun.

**Performance matters more than you think.** We started at 60fps and immediately regretted it. 12fps (80ms) looks perfectly smooth for terminal animation and keeps the CPU happy.

## Try It Yourself

```bash
# Clone and run
git clone https://github.com/isopath/matrix.go
cd matrix.go
go run .

# Or with options
go run . --options

# Or a specific file
go run . --file your-text.txt
```

The full code is on GitHub. Fork it, add your own text files, change the colors, make it yours. That's the beauty of open source.

---

**What's next?** We're thinking about image-to-ASCII with the same rain effect, or maybe a Star Wars crawl-style renderer. What would you build?
