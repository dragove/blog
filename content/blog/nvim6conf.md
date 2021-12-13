+++
title = "基于Lua的neovim配置指南"
date = 2021-12-13
[taxonomies]
tags=["neovim"]
archives=["2021"]
+++

Neovim 0.6 发布已经有一段时间了，在这里记录一下个人的配置记录。

## 目录结构

建议将 neovim 配置放到 `$HOME/.config/nvim` 目录下，neovim 默认会读取该目录下的 `init.lua` 文件加载配置。

```text
📂 ~/.config/nvim
├── 📁 ftplugin                 --> 按需加载的配置写在这里
├── 📂 lua                      --> 可以直接 require 加载的配置
│  ├── 🌑 myluamodule.lua
│  └── 📂 other_modules         --> 模块化的目录，底下必须有一个init.lua
│     ├── 🌑 anothermodule.lua
│     └── 🌑 init.lua
└── 🌑 init.lua
```

init.lua --> 配置加载起始目录

lua 目录 --> 详细配置目录，可以通过 `require` 函数直接加载的目录

ftplugin --> 文件类型配置，根据文件类型按需加载的配置写在这里。

## 个人配置

`~/.config/nvim/init.lua` 文件只有 require 调用，保持简洁，配置都写到 lua 目录下，lua 目录下的文件全部为 `init-xxx.lua`

```lua
-- init.lua
require('init-misc')
-- lua/init-misc.lua
vim.o.hidden = true -- 所有的 set xxx 操作变更为 vim.o.xxx = ?
vim.o.mouse = 'a' -- 用于使得鼠标可以在vim上操作
```

基础配置问题解决之后，我们需要引入包管理工具，用于下载、安装、使用网友编写的插件。这里推荐使用 [packer.nvim](https://github.com/wbthomason/packer.nvim)

```lua
-- init.lua 增加
require('init-packer')

-- lua/init-packer.lua
return require('packer').startup(function(use)
  -- package manager
  use 'wbthomason/packer.nvim'

  -- lua functions
  use 'nvim-lua/plenary.nvim'
  -- popup windows implementation
  use 'nvim-lua/popup.nvim'
  -- icons for other plugins
  use 'kyazdani42/nvim-web-devicons'
  -- termial integration
  use 'akinsho/nvim-toggleterm.lua'

  -- auto completion
  use 'hrsh7th/nvim-cmp'
  use 'hrsh7th/cmp-path'
  use 'hrsh7th/cmp-buffer'
  use 'hrsh7th/cmp-nvim-lsp'
  -- snippet support
  use 'L3MON4D3/LuaSnip'
  use 'saadparwaiz1/cmp_luasnip'
  -- lsp support
  use 'neovim/nvim-lspconfig'
  -- lsp UI staffs
  use 'tami5/lspsaga.nvim'
  -- java lsp
  use 'mfussenegger/nvim-jdtls'
  -- dap support
  use 'mfussenegger/nvim-dap'
  use 'rcarriga/nvim-dap-ui'
  use 'theHamsta/nvim-dap-virtual-text'
  -- treesitter config
  use 'nvim-treesitter/nvim-treesitter'
  -- show scope in code with treesitter
  use 'SmiteshP/nvim-gps'
  -- auto pairs
  use 'windwp/nvim-autopairs'
  -- auto tags
  use 'windwp/nvim-ts-autotag'
  -- surround with
  use 'blackCauldron7/surround.nvim'
  -- comment
  use 'numToStr/Comment.nvim'
  -- indent
  use 'lukas-reineke/indent-blankline.nvim'

  -- color scheme
  use 'NTBBloodbath/doom-one.nvim'
  -- status line
  use 'windwp/windline.nvim'
  -- clickable buffer line
  use 'akinsho/nvim-bufferline.lua'
  -- git integration
  use 'lewis6991/gitsigns.nvim'
  -- which-key
  use 'folke/which-key.nvim'


  -- file explorer
  use 'kyazdani42/nvim-tree.lua'
  -- fuzzy finder
  use 'nvim-telescope/telescope.nvim'
  -- media file preview extension for telescope
  use 'nvim-telescope/telescope-media-files.nvim'
end)
```

每个插件的配置均可以在插件的 github repo 上查阅。

