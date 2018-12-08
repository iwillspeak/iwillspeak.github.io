---
title: Racing Onwards
layout: post
published: true
---

I'm a long term Emacs user. When I first started writing rust I concocted an Emacs configuration using `racer-mode` and `rust-mode` directly which got the job done. Since then the state of Rust editor support has come a long way. I've finally taken the plunge and moved to RLS for a richer language completion experience.

RLS is the Rust Language Server. It implements the standardised "language server protocol"; which provides a common interface between editors and language completions. For Rust it uses a mixture of `racer` the Rust autocompletion tool and more heavyweight analysis.

Before setting things up you need to make sure you have a few toolchain components installed:  
  
```bash  
$ rustup component add rust-src rls-preview rust-analysis rustfmt-preview
```

The good news is that these should all be available on the `stable` toolchain right now.

Configuration wise I prefer to manage things with [`use-package`](https://jwiegley.github.io/use-package/). For a simple setup you should just need [`rust-mode`](https://github.com/rust-lang/rust-mode) and [`lsp-mode`](https://github.com/emacs-lsp/lsp-mode):  
  
```el
(use-package rust-mode
  :ensure t
  :mode "\\.rs"
  :init (add-hook 'rust-mode-hook 'lsp))
(use-package lsp-mode
  :ensure t
  :defer t
  :commands lsp
  :config
  (use-package lsp-clients))
```

Personally I prefer to use [Company](http://company-mode.github.io/) for completions. To switch to that all you ned to do is require the `company-lsp` mode.

```el
(use-package company-lsp
  :ensure t
  :after company)
```

That should be it. I like to keep my mode line neat and tidy so my full configuration contains a few `:diminish`es and re-maps some keys. Check out [the final result](https://github.com/iwillspeak/.emacs.d/blob/cd428dc25dfeac8795515ec75d3d8608db6c0879/init.el#L335). My old `racer-mode` config is there too, commented out for posterity. 