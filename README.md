# Today I Learned

My Today I Learned snippets. Inspired by [jbranchaud/til](https://github.com/jbranchaud/til) and [simonw/til](https://github.com/simonw/til)

## setup
```sh {til_setup.sh}
git clone https://github.com/hibariba/til.git
cd til
uv sync
source .venv/bin/activate
```
for running code snippets from markdown files install `runme`:
```sh
brew install runme
```

## usage

run code from markdown files:
```sh
runme list
# NAME        FILE               FIRST COMMAND  DESCRIPTION  NAMED
# client.py*  mcp/mcp-client.md  import mcp                  Yes
runme run client.py
```
