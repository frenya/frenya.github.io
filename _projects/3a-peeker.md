---
name: Peeker
tools: [vscode, typescript]
image: /assets/img/peeker.png
description: VSCode extension for easy lookup of currently selected text on a predefined website (e.g. a dictionary).
---

# Peeker

Peeker is a lightweight [VSCode extension](https://marketplace.visualstudio.com/items?itemName=frenya.vscode-peeker){:target="_blank"}
with a single purpose - quickly lookup currently selected text on a predefined website.

![Peeker](https://recall.frenya.net/assets/img/peeker.gif)

## How to use it

Open the Peeker panel by running the "Open Peeker" command
(easiest way is to press `Ctrl-P` to see list of commands and start typing the name).

After that, just select any text in any editor and corresponding content will be loaded.

## Configuration

Peeker only has one setting - `peeker.template` - which is a url string with `%s` sequence that will be replaced with the selected text.

For example, the above demo uses the string `https://www.urbandictionary.com/define.php?term=%s` which looks up terms in the urban dictionary.

## Disclaimer

The webview in VSCode is not a full-fledged browser. Clicking on links won't have any effect. Also, some websites may not load properly. Feel free to log an issue on GitHub if there's a specific site you'd like to be able to access.
