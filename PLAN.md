# PLAN — Two separate SSH projects

Host: Hackclub Nest Debian 13. Rule: different port than 22, keep host key, let anyone in, never give shell (Wish has none by default).
Workflow for both: code on Windows, `git push`, on Nest `git pull` + `go mod tidy` + `go run .`. No venv — `go.mod` is env.

## Project A — Infinite Story
Repo: https://github.com/HarvyLiu/The-Show-Must-Go-On
Folder: `C:\Users\spp_l\Downloads\The-Show-Must-Go-On`
Dev port `:23234`, prod `-p 2222`. Files: `.ssh/id_ed25519` + `story.json` (do not delete on Nest).
Goal: anyone ssh in, read last 10 lines, press `a` to add 1 sentence.

Stack: Go 1.27.1/1.24, `charm.land/wish/v2`, `charm.land/bubbletea/v2`, `lipgloss/v2`. `gopls` DONE.

State: `go.mod:1` has `wish/v2 v2.0.5 // indirect`, `main.go:1-8` hello + bare `NewServer()`.

Build:
- Data: `type Line{Author,Text}`, `var store{sync.Mutex; Lines[]Line}`, `load()` via `ReadFile+Unmarshal`, `save()` via `Marshal+WriteFile 0644` in `go save()`. Author=`s.User()` MVP.
- Server: `wish.NewServer(WithAddress, WithHostKeyPath)` + `ListenAndServe` + `bubbletea.Middleware(teaHandler)` + `activeterm/logging`.
- Tea model `{user,width,height,offset,writing,draft}`: `teaHandler` grabs `pty.Window`; `Update` handles `WindowSizeMsg`, `q|ctrl+c→Quit`, `a→writing`, `esc→cancel`, `enter→append+save`, `backspace`, `up/down`, `r`; `View` shows `last 10` + `> draft█`, `AltScreen=true`.
- Verify: `go run .` hangs, `ssh localhost -p 23234` works with 2 clients.

Ship: README top has exact ssh command, screenshot, how to connect/controls/how it works.

## Project B — SSH Aquarium
Repo: NEW — create `SSH-Aquarium` via `gh repo create SSH-Aquarium --public --source=. --push` in new folder `C:\Users\spp_l\Downloads\SSH-Aquarium` (run `go mod init ssh-aquarium` there).
Dev port `:23235`, prod `-p 2223` (must differ from Story). Files: own `.ssh/id_ed25519` + `aquarium.json`.
Goal: connect → get persistent fish named after you, swims forever, `f` to feed, arrows to nudge. Shared tank.

Data model:
```go
type Fish struct{
  ID string // fingerprint of s.PublicKey()
  Name string // s.User()
  X,Y float64; Vx,Vy float64
  Color, Symbol string // hash(ID)
  LastSeen time.Time
}
var world = struct{ sync.Mutex; Fishes []Fish; Foods []Food }{}
type Food struct{ X,Y float64; Life int }
```
- Load `aquarium.json` on boot, save every 50 ticks + on new fish via `go saveAquarium()`.
- New ID → random spawn speed 0.5-1.5. Existing → update Name/LastSeen, keep pos. Cap 50, prune >30d.

Tick loop:
- `Init()=Tick(100ms)`, `Update(TickMsg)`: `X+=Vx,Y+=Vy`, bounce at `0,W`, chase nearest Food, eat if dist<1, food falls `Y+=0.2,Life--`.
- Each viewer ticks but mutates shared world under lock, copy slice for View.

Render:
- MVP: sort by Y, each line `Repeat(" ",int(X)) + style.Render(Symbol+" "+Name)`, border via lipgloss, bubbles `~` from `(tick+hash)%W`.
- `WindowSizeMsg` → clamp `X,Y` to `W-10,H-5`. Color via `Foreground(hashColor)` (auto fallback).
- Header: `fishes N, press f`. Online count via `atomic.Int32` inc/dec on `sess.Context().Done()`.

Verify: 2 ssh clients see same movement, restart restores fishes, resize safe.

Ship: separate README with its own `ssh -p 2223` command, screenshot, controls.

## Go learning
- Story: `package/import/func`, `:= vs =`, `s,err`, `struct/slice/mutex`, `json`, `Model/Update/View`.
- Aquarium reuses all + adds `Tick`, `float64` physics, shared-world locking.
