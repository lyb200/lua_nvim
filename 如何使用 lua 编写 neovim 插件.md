# 如何使用 lua 编写 neovim 插件

Neovim 的开发人员给他们自己设定的目标之一是：使 lua 成为替代 vimL 的头等脚本语言。从版本 0.4 开始，其解释器和 'stdlib'(标准库) 就一起已经内置在编辑器中了。lua 非常容易学习，运行速度很快，广泛用于游戏开发社区中。在我看来，它的学习曲线也比 vimL 低得多，这可能会鼓励新人开始他们扩展 neovim 能力的旅程，或者只是为了一次性的目标制作简单的脚本。那么，让我们来尝试一下吧？我们编写一个简单的插件来显示我们最近使用过的文件。我们应该如何命名呢？也许...“我们已经编辑了什么 (What have I done，即缩写 whid)？！”。它看起来像这样：

![git](https://res.cloudinary.com/practicaldev/image/fetch/s--WWYGbh0u--/c_imagga_scale,f_auto,fl_progressive,h_420,q_auto,w_1000/https://2n.pl/system/blog_posts/photos/000/000/131/original/whid.png)

## 插件目录结构

我们的插件至少应该有两个目录：`plugin` 用于存放主文件，lua 目录放置整个基本代码。当然，如果需要的话，我们可以把所有代码都放在一个文件中，但是，请不要成为这样的人。因此，创建 `plugin/whid.vim` 和 `lau/whid.lua` 比较好。我们从这里开始：

```vim
" in plugin/whid.vim
" prevent loading file twice, 防止文件加载两次
if exists('g:loaded_whid.vim') | finish | endif

let s:save_cpo = &cpo      "save user coptions 保存用户的 coptions
set cpo&vim                " reset them to defaults, 把它们重置为默认

" command to run our plugin, 创建用于运行插件的命令
command! whid lua require'whid'.whid()

let &cpo = s:save_cpo       " and restore after, 恢复选项
unlet s:save_cpo

let g:loaded_whid = 1
```

`let s:save_cpo = &cpo` 是一种常用的操作，以防止自定义的 `coptions` （单个字符标志组成的序列）选项干扰插件的功能。对我们的目的而言，该行的缺失可能不会有什么不良后果，但是这被认为是一种良好的实践（至少按照 vim 帮助文件是这样的）。`command! whid lua require'whid'.whid()` ，其要求插件的 lua 模块且用其主函数。

## 浮动窗口

好吧，让我们从有趣的事件开始。我们应该创建一个可以显示内容的地方。感谢 neovim（现在 vim 也可以）有称为浮动窗口的功能。它是一个显示在其他窗口之上的窗口，与 OS（操作系统）一样。

```lua
-- in lua/whid.lua

local api = vim.api                      -- 这是为了后面编程简单点
local buf, win                           -- 定义两个空的局部变量

local function open_window()
  -- create new empty buffer，创建新的空缓冲区
  buf = api.nvim_create_buf(false, true)
  -- 给buf设置选项，即当缓冲区隐藏时予以清除
  api.nvim_buf_set_option(buf, 'bufhidden', 'wipe')

  -- get dimensions，获得编辑器编辑窗口的尺寸
  local width = api.nvim_get_option('columns')
  local height = api.nvim_get_option('lines')

  -- calculate our floating window size，计算浮动窗口的大小
  local win_width = math.ceil(width * 0.8)
  local win_height = math.ceil(height * 0.8 - 4)

  -- and its starting position，和浮动窗口的起始位置
  local row = math.ceil((height - win_height) / 2 - 1)
  local col = math.ceil((width - win_width) / 2)

  -- set some options，给缓冲区设置一些选项
  local opts = {
    style = 'minimal',
    relative = 'editor',
    width = win_width,
    height = win_height,
    row = row,
    col = col
  }

  -- and finally create it with buffer attached
  -- 最后使用与之相连的缓冲区创建窗口
  win = api.nvim_open_win(buf, true, opts)
end
```

在文件的顶部，我们定义了两个具有最高作用域的变量：`win` 和 `buf` ，它们经常会被其他函数引用。此时，变量的值是空的，该缓冲区将成为我们摆放我们查询结果的地方。它在创建时提供以下选项：缓冲区不会被在编辑器使用 `:ls` 列表显示出来（第一个参数）和 “临时缓冲区(scratch-buffer)”（第二个参数，参见 `:h scratch-buffer`）。同时，我们设置，当缓冲区被隐藏时 `bufhidden = wipe` ，窗口会被删除。

使用 `nvim_open_win(buf, true, opts)` ，就能让我们使用前面已创建的相关联的缓冲区来创建新窗口。第二个参数使新窗口获得焦点。`width` 和 `height` （宽和高）的含义不言自明。`row` 和 `col` （行和列）是窗口的起始位置，是基于编辑器 `relative = "editor"` 左上角计算而来。`style = "minimal"` 是一个用于配置窗口外观的方便选项，这里我们禁用了许多不想要的选项，例如行号或者拼写错误的高亮。

现在，我们已经有了浮动窗口，但是我们可以使它更好看。目前 neovim 不支持像边框这样的小部件，因此我们应该自己创建一个。这相当简单。我们需要另一个浮动窗口，稍微比第一个大一点，并把该窗口放置在下面。

```lua
local border_opts = {
  style = 'minimal',
  relative = 'editor',
  width = win_width + 2,
  height = win_height + 2,
  row = row -1,
  col = col -1
}
```

我们将用“方框绘画”字符串填充它。（译者：第二个窗口比第一个大点，只有边框部分填充相应的字符，中间是空的，并放置在第一个窗口下面，所以整体看起来就是一个有边框的浮动窗口。）

```lua
local border_buf = api.nvim_create_buf(flase, true)

-- 译者：以下生成这个方框，先生成一个数组
local border_lines = { '╔' .. string.rep('═', win_width) .. '╗' }
local middle_line = '║' .. string.rep(' ', win_width) .. '║'
for i = 1, win_height do
  table.insert(border_lines, middle_line)
end
table.insert(border_lines, '╚' .. string.rep('═', win_width) .. '╝')

-- 译者：使用上面创建的方框数组来填充这个缓冲区
api.nvim_buf_set_lines(border_buf, 0, -1, false, border_lines)
-- set buffer's (border_buf) lines from first line (0) to last (-1)
-- 设置缓冲区的行，从第一行 (0) 到最后行 (-1)
-- ignoring out-of-bounds error (false) with lines (border_lines)
-- 在使用行（border_lines）填充时，忽略边界溢出错误（参数：false）
```

当然，我们必须按照正确的次序打开这两个窗口。另外，两个窗口应该同时关闭。如果第一个窗口关闭后，边框缓冲区还在哪里，这会是非常奇怪。目前 vimL 的自动命令是解决这个问题的最佳方案。

```lua
-- 译者：创建方框浮动窗口和主窗口
local border_win = api.nvim_open_win(border_buf, true, border_opts)
win = api.nvim_open_win(buf, true, opts)
-- 译者：设置自动命令，当主窗口关闭时触发安静地执行方框窗口也关闭
api.nvim_command('au BufWipeout <buffer> exe "silent bwipeout! "' .. border_buf)
```

## 获取数据

这个插件的设计目的是显示最近我们编辑过的文件。我们将使用简单的 `git` 命令来演示。像这样：

```lua
git diff-tree --no-commit-id --name-only -r HEAD
```

让我们创建一个函数，它会在我们漂亮的窗口中放入一些数据。我们将经常调用它，所以，把它命名为：`update_view` 。

```lua
local function update_view()
  -- we will use vim systemlist function which run shell
  -- command and return result as list
  -- 使用能运行 shell 命令，且以列表形式返回结果的 vim 的 systemlist 函数
  local result = vim.fn.systemlist('git diff-tree --no-commit-id --name-only -r HEAD')

  -- with small indentation results will look better
  -- 使用较小缩进，结果看起来更好看
  for k, v in pairs(result) do
    result[k] = '  ' .. result[k]
  end

  -- 译者：把结果填充到缓冲区中
  api.nvim_buf_set_lines(buf, 0, -1, result)
end
```

嗯...，如果我们只查看当前的这些文件，这会是非常不方便的。因为我们会直接调用这个函数来更新我们的视图，我们应该接受参数，而该参数表示我们想要显示被编辑过的文件的时间远点还是更近点状态的信息。

```lua
local position = 0

local function update_view(direction)
  position = position + direction
  -- HEAD~0 is the newest state，HEAD~0表示是最新的状态
  if position < 0 then position = 0 end

  local result = vim.fn.systemlist('git diff-tree --no-commit-id --name-only -r  HEAD~'..position)

  -- ... rest of the code，代码剩余部分
```

你知道这里还缺少什么吗？我们插件的标题。而且要居中！该函数将帮助我们在中间放置文本。

```lua
local function center(str)
  local width = api.nvim_win_get_width(0)
  local shift = math.floor(width / 2) - math.floor(string.len(str) /2)
  return string.rep(' ', shift) .. str
end
```

```lua
  api.nvim_buf_set_lines(buf, 0, -1, false, {
    center('What have I done?'),
    center('HEAD~' .. position),
    ''   -- 译者：空一行
  })
end
```

如果有高亮显示更好。有几种选项我们可以选择：自定义语法文件（你可以匹配基于行号的模式），或者使用虚拟文本注释，而不是普通文件（这是可能的，但是这样我们的代码会更复杂），或者......。我们可以使用基于高亮`nvim_buf_add_highlight` 的位置。但是，首先我们得定义我们的高亮。我们会链接到默认的高亮组，而不是我们自己设置的颜色。以下方法会匹配到我们的颜色主题。

```vim
" in plug/whid.vim after set cpo&vim

hi def link whidHeader      Number
hi def link whidSubHeader   Identifier
```

现在加入高亮：

```lua
api.nvim_buf_add_highlight(buf, -1, 'whidHeader', 0, 0, -1)
api.nvim_buf_add_highlight(buf, -1, 'whidSubHeader', 1, 0, -1)
```

我们给缓冲区 `buf` 增加高亮，而该高亮是没有分组的高亮（第二个参数 `-1` ）。我们可以在第二个参数这里传递命名空间的 id 值，这样我们就可以同时清除组内所有高亮，但是在我们的案例中不需要这样做。接下来是行号和最后两个参数是列范围的开始和结束（字节索引）位置。

整个函数看起来像这样：

```lua
local function update_view(direction)
  -- Is nice to prevent user from editing interface, so
  -- we should enabled it before update view and disableed after it.
  -- 防止用户使用编辑接口是好的，因此我们应该在更新试图前启用浮动窗口的编辑功能，
  -- 且在结束后禁用浮动窗口的编辑功能。
  api.nvim_buf_set_option(buf, 'modifiable', true)

  -- 译者：计算要显示编辑过的文件时间远近
  position = position + direction
  if position < 0 then position = 0 end

  -- 译者：获取相应的编辑过的文件集合
  local result = vim.fn.systemlist('git diff-tree --no-commit-id --name-only -r  HEAD' .. position)
  -- 译者：在获取的文件前面加两个空格
  for k, v in pairs(result) do
    result[k] = '  ' .. result[k]
  end

  -- 译者：把标题填充到窗口中
  api.nvim_buf_set_lines(buf, 0, -1, false, {
    center('what have I done?'),
    center('HEAD~' .. position),
    ""
  })
  -- 译者：把结果填充到窗口中
  api.nvim_buf_set_lines(buf, 3, -1, false, result)

  -- 译者：加入高亮
  api.nvim_buf_add_highlight(buf, -1, 'whidHeader', 0, 0, -1)
  api.nvim_buf_add_highlight(buf, -1, 'whidSubHeader', 1, 0, -1)

  -- 译者：禁用浮动窗口的编辑功能
  api.nvim_buf_set_option(buf, 'modifiable', false)
end
```

## 用户输入

现在我们应该使我们的插件具有互动性。这不会太复杂，只是一些简单的功能，例如改变当前的预览状态，或选择一个文件且打开它。插件会接受到用户通过键映射的输入。即点击一个键会触发某些动作。让我们看看在 `lua api` 中如何定义映射。

```lua
api.nvim_buf_set_keymap(buf, 'n', 'x', ':echo "wow!"<cr>', {nowait = true, noremap = true, silent = true})
```

第一个参数通常是缓冲区。这些映射只在该缓冲区中起作用。下一个是模式名称的缩写。我们定义所有的映射都是常规模式 `n` 。然后是一个“左侧”的键绑定（我选择 `x` 作为示例）映射到“右侧”的键绑定（我们告诉 neovim 进入命令行模式，输入一些 vimL 命令，然后点击回车）。最后是一些选项。我们要 neovim 在匹配到模式时立即触发映射，因此我们设置 `nowait` 标记，我们要防止它通过其他映射触发我们的映射，需要选项 neremap ，且不会给用户显示输入的内容，选项 silent。语句太长，因此我们使用数组来节省一些代码的书写。

```lua
local function set_mappings()
  local mappings = {
    ['['] = 'update_view(-1)',
    [']'] = 'update_view(1)',
    ['<cr>'] = 'open_file()',
    h = 'update_view(-1)',
    l = 'update_view(1)',
    q = 'close_window()',
    k = 'move_cursor()',
  }

  for k, v in pairs(mappings) do
    api.nvim_buf_set_keymap(buf, 'n', k, ':lua require"whid".'..v..'<cr>', {
      nowait = true, noremap = true, silent = true
    })
  end
end
```

我们也可以禁用那些不使用的键（或者不禁用，随你喜欢）。

```lua
local othe_chars = {
  'a', 'b', 'c', 'd', 'e', 'f', 'g', 'i', 'n', 'o', 'p', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z'
}
for k, v in pairs(othe_chars) do
  -- 译者：分别禁用小写、大写和 与 ctrl 的组合键
  api.nvim_buf_set_keymap(buf, 'n', v, '', { nowait = true, noremap = true, silent = true })
  api.nvim_buf_set_keymap(buf, 'n', v:upper(), '', { nowait = true, noremap = true, silent = true })
  api.nvim_buf_set_keymap(buf, 'n', '<c-'..v..'>', '', { nowait = true, noremap = true, silent = true })
end
```

## 公共函数

映射设置好了，但是在这些映射中还有一些新的函数。让我们看看它们。

```lua
local function close_window()
  api.nvim_win_close(win, true)
end

-- Our file list start at line 4, so we can prevent reaching above it
-- from bottom the end of the buffer will limit movement
-- 文件列表始于第4行，因此我们可以防止光标到达到第4行之上，
-- 从底部向上到达缓冲区末端会限制移动
local function move_cursor()
  local new_pos = math.max(4, api.nvim_win_get_cursor(win)[1] - 1)
  api.nvim_win_set_cursor(win, {new_pos, 0})
end

-- open file under cursor 打开光标下的文件
local function open_file()
  local str = api.nvim_get_current_line()
  close_window()
  api.nvim_command('edit' .. str)
end
```

我们的文件列表非常简单。我们仅仅是获取光标下的行内容，然后告诉 neovim 编辑它。当然，我们可以创建更复杂的机制。我们可以得到行号（或者甚至列），然后基于得到的，触发特殊的动作。它将允许分离视图和逻辑。但是对于我们的目的来说，这已经够了。

但是，如果我们没有先暴露这些函数，它们都不能被调用。在文件的结尾，我们会返回带有公共可用函数的关联数组。

```lua
return {
  whid = whid,
  update_view = update_view,
  open_file = open_file,
  move_cursor = move_cursor,
  close_window = close_window
}
```

并且，当然还有主函数！

```lua
local function whid()
  -- if you want to preserve last displayed state just omit this line
  -- 如果你想保留最后显示的状态，只需省略这一行
  position = 0
  open_window()
  set_mappings()
  update_view(0)
  -- set cursor on first list entry
  -- 把光标设置到列表的第一个条目上
  api.nvim_win_set_cursor(win, {4, 0})
end
```

## 插件所有代码 ...

... 有很小的修改完善（你可以通过查看注释找到它们）。

```vim
" plug/whid.vim

if exists('g:loaded_whid') | finish | endif

let s:save_cpo = &cpo
set cpo&vim

hi def link whidHeader      Number
hi def link whidSubHeader   Identifier

command! whid lua require'whid'.whid()

let &cpo = s:save_cpo
unlet s:save_cpo

let g:loaded_whid = 1
```

```lua
-- lua/whid.lua

local api = vim.api
local buf, win
local position = 0

local function center(str)
  local width = api.nvim_win_get_width(0)
  local shift = math.floor(width / 2) - math.floor(string.len(str) / 2)
  return string.rep(' ', shift) .. str
end

local function open_window()
  buf = api.nvim_create_buf(false, true)
  local border_buf = api.nvim_create_buf(flase, true)

  api.nvim_buf_set_option(buf, 'bufhidden', 'wipe')
  api.nvim_buf_set_option(buf, 'filetype', 'whid')

  local win_height = math.ceil(height * 0.8 - 4)
  local win_width = math.ceil(whidth * 0.8)
  local row = math.ceil((height - win_height) / 2 -1)
  local col = math.ceil((width - win_width) / 2)

  local border_opts = {
    style = "minimal",
    relative = "editor",
    width = win_width + 2,
    height = win_height + 2,
    row = row - 1
    col = col -1
  }

  local opts = {
    style = "minimal",
    relative = "editor",
    width = win_width,
    height = win_height,
    row = row,
    col = col
  }

  local border_lines = { '╔' .. string.rep('═', win_width) .. '╗' }
  local middle_line = '║' .. string.rep(' ', win_width) .. '║'
  for i=1, win_height do
    table.insert(border_lines, middle_line)
  end
  table.insert(border_lines, '╚' .. string.rep('═', win_width) .. '╝')
  api.nvim_buf_set_lines(border_buf, 0, -1, false, border_lines)

  local border_win = api.nvim_open_win(border_buf, true, border_opts)
  win = api.nvim_open_win(buf, true, opts)
  api.nvim_command('au BufWipeout <buffer> exe "silent bwipeout! "'..border_buf)

  -- It highlight line with the cursor on it
  -- 设置选项使光标所在行高亮显示
  api.nvim_win_set_options(win, 'cursorline', true)

  -- we can add title already here, because first line will never change
  -- 我们可以在这里加上标题，因为第一行永远不会改变
  api.nvim_buf_set_lines(buf, 0, -1, false, { center('what have I done?'), '', ''})
  api.nvim_buf_add_highlight(buf, -1, 'whidHeader', 0, 0, -1)
end

local function update_view(direction)
  api.nvim_buf_set_option(buf, 'modifiable', true)
  position = position + direction
  if position < 0 then position = 0 end

  local result = vim.fn.systemlist('git diff-tree --no-commit-id --name-only -r  HEAD~'.. position))
  -- add  an empty line to preserve layout if there is no results
  -- 如果没有结果，则添加一个空行以保留布局
  if #result == 0 then table.insert(result, '') end
  for k, v in pairs(result) do
    result[k] = '  '..result[k]
  end

  api.nvim_buf_set_lines(buf, 1, 2, false, {center('HEAD~'..position)})
  api.nvim_buf_set_lines(buf, 3, -1, false, result)

  api.nvim_buf_add_highlight(buf, -1, 'whidSubHeader', 1, 0, -1)
  api.nvim_buf_set_option(buf, 'modifiable', false)
end

local function close_window()
  api.nvim_win_close(win, true)
end

local function open_file()
  local str = api.nvim_get_current_line()
  close_window()
  api.nvim_command('edit '.. str)
end

local function move_cursor()
  local new_pos = math.max(4, api.nvim_win_get_cursor(win)[1] -1)
  api.nvim_win_set_cursor(win, {new_pos, 0})
end

local function set_mappings()
  local mappings = {
    ['['] = 'update_view(-1)',
    [']'] = 'update_view(1)',
    ['<cr>'] = 'open_file()',
    h = 'update_view(-1)',
    l = 'update_view(1)',
    q = 'close_window()',
    k = 'move_cursor()'
  }

  for k,v in pairs(mappings) do
    api.nvim_buf_set_keymap(buf, 'n', k, ':lua require"whid".'..v..'<cr>', {
        nowait = true, noremap = true, silent = true
      })
  end

  local other_chars = {
    'a', 'b', 'c', 'd', 'e', 'f', 'g', 'i', 'n', 'o', 'p', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z'
  }
  for k,v in ipairs(other_chars) do
    api.nvim_buf_set_keymap(buf, 'n', v, '', { nowait = true, noremap = true, silent = true })
    api.nvim_buf_set_keymap(buf, 'n', v:upper(), '', { nowait = true, noremap = true, silent = true })
    api.nvim_buf_set_keymap(buf, 'n',  '<c-'..v..'>', '', { nowait = true, noremap = true, silent = true })
  end
end

local function whid()
  position = 0
  open_window()
  set_mappings()
  update_view(0)
  api.nvim_win_set_cursor(win, {4, 0})
end

return {
  whid = whid,
  update_view = update_view,
  open_file = open_file,
  move_cursor = move_cursor,
  close_window = close_window
}
```

现在您应该已经掌握了一些基本知识，足以为您的 `lua neovim` 脚本编写简单的 TUI（文本型用户接口）！

[这些代码也可以在这里找到](https://github.com/rafcamlet/nvim-whid)

---

原文：[How to write neovim plugins in Lua](https://dev.to/2nit/how-to-write-neovim-plugins-in-lua-5cca)
