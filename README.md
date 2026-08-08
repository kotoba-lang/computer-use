# computer-use-clj

Anthropic-computer-use-style desktop automation agent in **portable
Clojure** — every namespace is `.cljc`, designed for
**Clojure-on-WASM hosts** (SCI, ClojureScript, GraalVM, kotoba-clj) as
well as the JVM. The desktop itself is an injected host capability;
the action history is persisted through a **Datomic API**.

Built on [langgraph-clj](https://github.com/com-junkawasaki/langgraph-clj)
/ [langchain-clj](https://github.com/com-junkawasaki/langchain-clj).
Sibling of [browser-use-clj](https://github.com/com-junkawasaki/browser-use-clj).

```
src/computeruse/
  computer.cljc  IComputer protocol (host capability) + mock virtual screen
  tool.cljc      the computer tool — Anthropic action vocabulary → protocol dispatch
  agent.cljc     sampling loop (langgraph StateGraph) + action log as datoms
```

## Design

- **Computer = injected host capability** — implement `IComputer`
  (xdotool/screencapture, a VNC sandbox, an OS-automation MCP).
  Screenshots are passed through untouched, so a real host can return
  Anthropic image content blocks while the bundled `mock-computer`
  returns a deterministic text rendering of a virtual screen.
- **Anthropic action vocabulary** — the `computer` tool speaks
  screenshot / key / type / mouse_move / left_click / right_click /
  middle_click / double_click / scroll / cursor_position, defined as a
  custom tool with an explicit `input_schema` (API-compatible without
  the server-defined tool type; hosts that want
  `{:type "computer_20250124" …}` can send that wire format and reuse
  `computeruse.tool/dispatch`).
- **Datomic API premise** — every action becomes a datom
  (`:caction/action`, `:caction/input`, …): "every click in session
  s1" is a Datalog query. Graph checkpoints (resume /
  human-in-the-loop) come from langgraph-clj.

## Quickstart

```clojure
(require '[computeruse.computer :as c]
         '[computeruse.agent :as agent]
         '[langchain.model :as model]
         '[langchain.db :as db])

;; host capability: real IComputer impl, or the mock virtual screen:
(def vm (c/mock-computer
         {:size [1280 800]
          :windows [{:title "Editor" :rect [0 0 800 600] :content ""}
                    {:title "Terminal" :rect [800 0 480 600] :content "$ "}]}))

(def conn (db/create-conn agent/log-schema))

(agent/run
 {:model (model/anthropic-model {:api-key … :http-fn host-fetch …})
  :computer (:computer vm)
  :display {:width 1280 :height 800}
  :task "Run make test in the terminal"
  :history-conn conn
  :session-id "s1"
  :max-steps 25})
;; => {:result "…" :done true :messages […] :steps n}

;; the audit trail is datoms:
(db/q '[:find ?step ?action
        :where [?e :caction/step ?step] [?e :caction/action ?action]]
      (db/db conn))
```

Extra tools (bash, editors, …) sit alongside the computer tool:

```clojure
(agent/run {:tools [my-bash-tool my-editor-tool] …})
```

## Mapping from upstream

See [docs/adr/0001-architecture.md](docs/adr/0001-architecture.md) for
the computer-use → computer-use-clj correspondence (action vocabulary,
sampling loop, tool wire format) and the host-capability split.

## Credentials from a vault (no raw secrets)

The agent never types a literal password. `computeruse.vault/IVault`
is an injected host capability; the `computer` tool gains a
`type_secret` action that takes a vault *reference*
(`{"item":"Vultr","field":"password"}` or `"op://Vault/Item/field"`),
resolves it through the vault CLI, and types it at the IComputer layer.
The secret never enters the prompt, the message history, or the
Datomic action log (which records only the ref).

```clojure
(require '[computeruse.vault :as vault])
(agent/run {:vault (vault/op-vault {})        ; 1Password: `op` signed in
            ;; or (vault/bw-vault {})         ; Bitwarden: BW_SESSION set
            :computer (macos/macos-computer)
            :system "...use type_secret for credential fields; never type a raw secret..."
            ...})
```

- `op-vault` → `op read op://Vault/Item/field` or `op item get … --reveal`
- `bw-vault` → `bw get password|username|totp <item>` (+ custom fields)
- `mock-vault` → deterministic map for tests

## Local model (Ollama / OpenAI-compatible)

`computeruse.openai-model/openai-model` drives the agent with a LOCAL
model instead of the Anthropic API — vision via image_url, tool-calling
via OpenAI `tools`. `examples/jvm_host.clj` provides the JVM host caps
(an :http-fn that shells out to `curl`, JSON via data.json).

```sh
# Gemma 4 E4B QAT on Ollama, controlling this desktop:
OPENAI_BASE_URL=http://127.0.0.1:11434/v1 OPENAI_MODEL=gemma4:e4b-it-qat \
VAULT=op VULTR_VAULT_ITEM=Vultr \
  clojure -M:gemma -m vultr-ip-allow 203.0.113.7 32
```

Notes from running Ollama 0.30.x: tool params of `type:"object"` must
include `properties` (a bare object 404s the whole request); tool-call
`arguments` must be a JSON string (not EDN); java.net.http POSTs were
rejected by Ollama's request framing, so the host :http-fn uses curl.
Small models (4-8B) drive navigation/keys/screenshots but are often too
weak to complete multi-step portal UIs reliably — use a larger model or
the Anthropic backend (`MODEL=anthropic`) for hard tasks.

## Real host: macOS

`computeruse.macos/macos-computer` implements IComputer over the live
macOS desktop (screencapture + sips screenshots as Anthropic image
blocks, System Events keys, cliclick mouse — `brew install cliclick`,
grant Screen Recording + Accessibility). Coordinates are auto-scaled
between the model-sized screenshot and display points.

`examples/vultr_ip_allow.clj` uses it for a real ops task — adding an
IP to a Vultr API key's Access Control list in an already-signed-in
browser session. Hard guardrails in the system prompt: the agent never
types into credential fields and bails out (success=false) when a
login/2FA page appears; it only ever adds the one requested entry.

`examples/sumitclub_meisai.clj` is a read-only variant — fetching a
card 利用明細 (statement) from sumitclub.jp. Login goes through
`type_secret` (vault ref, never a raw credential), the system prompt
forbids every state-changing control on the site, and the extracted
rows are persisted via a custom `save_statement` tool as EDN, ready
for downstream ingestion. It runs on a **local model by default**
(Ollama serving gemma 4 QAT — tools + vision capable), so statement
data never leaves the machine; `examples/jvm_host.clj` provides the
JVM host capabilities and the `LLM=ollama|gemini|anthropic` switch
(gemini = Gemini's OpenAI-compatible endpoint with `GEMINI_API_KEY`).

`examples/ngc_free_org.clj` prepares the frontmost browser for a free
individual NVIDIA NGC organization registration:

```sh
clojure -M:dev:examples -m ngc-free-org
```

The `computeruse.ngc/free-registration-system-prompt` is intended for an
AI-driven inspection loop. It prohibits credential entry, MFA, terms
entry, MFA, terms acceptance, and account creation until it has called the
shared `request_human_approval` tool. The macOS host presents this request as a
native alert containing only the summary, action, and external impact:

```sh
LLM=ollama clojure -M:examples -m ngc-free-org-agent
```

```sh
SUMITCLUB_VAULT_ITEM=sumitclub \
  clojure -M:dev:examples -e "(require 'sumitclub-meisai) (sumitclub-meisai/-main)"
```

```sh
ANTHROPIC_API_KEY=… clojure -Sdeps '{:paths ["src" "examples"]
                 :deps {io.github.kotoba-lang/langgraph
                        {:git/sha "a332a770a0d2b5193f81b54483bb954fb29ef8d7"}}}' \
        -M -e "(require 'vultr-ip-allow) (vultr-ip-allow/-main \"203.0.113.7\")"
```

## Real host: a physical iPhone, through iPhone Mirroring

`computeruse.ios-mirroring/iphone-mirroring-computer` implements IComputer
over a **physical iPhone** as macOS 15+ mirrors it. There is no `adb` for
iOS — Appium / WebDriverAgent / XCUITest are not here, and `simctl` is
simulator-only and has no `tap` — so the channel is the mirroring window
itself: `screencapture -R` over its rect for the screenshot, `cliclick`
inside it for taps and swipes, System Events for keys.

```clojure
(ios/iphone-mirroring-computer {:model-width 480})
```

Two corrections distinguish it from the Android driver, and both are silent
when wrong, so both are pinned by tests: the model's coordinates are offsets
into a **crop**, so the content origin has to be added back; and the scale is
points-per-model-pixel, taken from the window rect in **points**, never from
the Retina capture's pixels.

`key "home"` / `"appswitcher"` / `"spotlight"` map to iPhone Mirroring's own
`⌘1` / `⌘2` / `⌘3`, which is how the phone is navigated — there is no home
button to tap. `right_click` is a press-and-hold, iOS's actual secondary
gesture. Swipes send interpolated motion samples in one `cliclick`
invocation; a press and a release with nothing between them is read by iOS as
a tap.

It cannot do multitouch (one pointer), cannot be frame-accurate (the picture
is a live video stream), and stops the moment someone picks the phone up.
Turn-based and slow real-time games are the honest target.

`examples/iphone_game_agent.clj` drives it as a game player:

```sh
clojure -M:dev:gemma -e "(require 'iphone-game-agent) (iphone-game-agent/-main \"Solitaire\")"
```

The pure half — coordinate mapping, gesture paths, key scripts — is tested
(`test/computeruse/ios_mirroring_test.cljc`, runs on cljs too). The shell-outs
are not, and cannot be without a Mac, a paired iPhone and two granted
permissions.

## Tests / example

```sh
clojure -M:test     # 30 tests, 121 assertions
clojure -Sdeps '{:paths ["src" "examples"]
                 :deps {io.github.kotoba-lang/langgraph
                        {:git/sha "a332a770a0d2b5193f81b54483bb954fb29ef8d7"}}}' \
        -M -e "(require 'desktop-agent) (desktop-agent/-main)"
```

Workspace development against local checkouts: `clojure -M:dev:test`.
