---
layout: post
title:  "Language Server Protocol"
description: An introduction to language server protocol.
img:
date: 2018-06-04  +1020
---

# Language Server Protocol
## What is it?
+ Started by Microsoft in 2015 (took me 3 years to hear about it).
+ De-couple language support from IDE by splitting into IDE-agnostic language server and language-agnostic IDE client.
+ Language developers write server to handle language related functionality of find definition, find references, code completion ...
+ IDE developers write client to handle IDE related functionality such as display definition on hovering, providing completion menu, display diagnostics in some pop-up menu ...

## Why?
+ Implements modularity and microservices (to some extent)
+ Increased code reusability.
+ Converts m\*n problem into m+n problems (just like compilers and architectures and intermediate representation).
+ Can use your favorite editor(Vi/Emacs) for all languages. I've faced this problem when I've been forced to use IntelliJ/Eclipse for Java and even with a Vim plugin, it is not Vim, it is just a shadow. So now, hopefully, if I figure out how to use Vim + Java LS, I can get an amazing setup for using Java in Vim.

## Why not?
+ Increased communication could cause some latency issues (noticed when trying Eclipse + Cquery).
+ It is only as good as the server (so cannot really replace IntelliJ with Emacs + Java Language Server, for example, since IntelliJ's intellisense has not been reproduced yet).
+ Bounded by the protocol => cannot get some language specific features like auto including dependencies etc.

# Thank you LSP!
I was recently working on something related to language servers. To test stuff out and to get some idea of the working of language servers (Cquery), I decided to try it out with Emacs since:
+ It was easily available on Windows.
+ There were some nice tools like [emacs-lsp](https://github.com/emacs-lsp/lsp-mode) and [emacs-cquery](https://github.com/cquery-project/emacs-cquery) which make using cquery with emacs just 2 package installs away.
+ The result was spectacular. It is just beautiful.
+ The list of packages that need to be installed for support are `company-lsp`, `lsp-ui`, `lsp-mode`
+ Here is what needs to be added to your `.emacs` file to get nice lsp support:

```
;; In general wrt to lsp, autocompletion and snippets
(require 'lsp-mode)

(require 'lsp-ui)
(add-hook 'lsp-mode-hook 'lsp-ui-mode)

(require 'company-lsp)
(push 'company-lsp company-backends)
(setq company-transformers nil company-lsp-async t company-lsp-cache-candidates 'auto company-lsp-enable-snippet t)

(require 'yasnippet)
(yas-global-mode 1)

;; Specific to cquery
(require 'cquery)
(setq cquery-executable "/path/to/cquery/build/release/bin/cquery")
(setq cquery-sem-highlight-method 'font-lock)
(setq cquery-extra-init-params '(:index (:comments 2) :cacheFormat "msgpack"))
(cquery-use-default-rainbow-sem-highlight)

```

+ To enable it, use:

```
M-x lsp-cquery-enable
M-x company-mode
```

Guess that needs to be automated.
