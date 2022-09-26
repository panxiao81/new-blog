---
title: 叛逃至 Neovim 
date: 2022-09-27T00:54:15+08:00
categories: ['Programming', 'Linux']
tags: ['programming','linux']
---

终于有一天，我的 vscode 因为装了大量插件他终于速度慢的我有点无法忍受了。

尤其是在我这台可怜的 Surface Go 2 上，每次打开都慢的要命不说，使用起来也非常卡顿。

偶然听说 Neovim 这个现代化的 Vim 编辑器，并且有良好的插件生态，所以我打算尝试一下。

<!--more-->

推动我进行这个迁移的其实是我偶然发现我有使用 Github Copilot 的资格。然后在支持的编程器中发现可以使用 Neovim。另外在 Youtube 上看到了一个从零开始自己配置 Neovim 的教程，因此催生了这个想法。

我配置了如下的插件：

- packer.nvim 作为插件管理器
- bufferline.nvim 状态栏插件，现代 Neovim 几乎人手一个的插件
- lualine.nvim Status Line 插件，也是几乎人手一个的插件
- Telescope 著名的搜索插件
- nvim-tree.lua 一个完全使用 Lua 编写的文件管理器插件
- nvim-treesitter 代码高亮，替代 Vim 自带的正则式高亮
- Gitsigns Git 集成
- toggleterm.nvim 替代 Neovim 自带终端的插件

整套补全系统没有使用 coc.nvim 而是使用了自带的 LSP，使用了一些插件作为进行代码补全和 Snippet 管理

- nvim-cmp 代码补全框架
- Luasnip Snippet 管理
- cmp_luasnip 将 Luasnip 接入 nvim-cmp
- nvim-lspconfig 官方的 LSP 配置插件
- nvim-lsp-installer 一个简单安装管理 LSP 服务器端的插件
- cmp-nvim-lsp 将 LSP 接入 nvim-cmp
- copilot.lua 一个第三方实现的 Github Copilot 客户端
- copilot-cmp 可将 copilot 接入 nvim-cmp，需要配合作者的 copilot.lua 使用

各位想抄配置的可以去看 [panxiao81/dotfiles](https://github.com/panxiao81/dotfiles)

尽管我的 Neovim 是装在 WSL 中，但在 Surface Go 2 上无论是响应速度还是打开的速度都非常的快，有过去使用 Sublime Text 的感觉，加上 LSP 完全可以写程序代码。

WSL 里套 Neovim 的一个大问题是剪切板不共享。我想使用 dotfiles 统一管理，而我由不仅仅只有 Windows 平台，因此我找到了一个 Neovim 的外壳程序 Neovide，他支持 WSL 集成，并且修好了我的剪切板。

多数 Nerd 字体没有考虑 CJK 文字的问题，我使用的字体是 Sarasa Mono SC Nerd，他严格保证了中文和拉丁字母 1:2 的排版，但总觉得字形不是很好看，如果有其他字体欢迎推荐给我。

或许有空会写配置，反正未来短期内没啥时间，所以自己去抄吧，反正我也是从各种地方抄来的。

