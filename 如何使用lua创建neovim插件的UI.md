# 如何使用 lua 创建 neovim 插件的用户界面

在上一篇 [文章](https://github.com/lyb200/lua_nvim/blob/main/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%20lua%20%E7%BC%96%E5%86%99%20neovim%20%E6%8F%92%E4%BB%B6.md) 中，我们学习了在 lua 中使用浮动窗口创建插件的基本知识。现在是时候采用更传统的方法了。让我们创建一个简单的插件，在方便的侧导航区显示最近打开过的文件。由于我们专注于学习该界面，因此我们将使用 vim 本地的打开过的文件列表(oldfiles)来实现这个目标。它看起来像这样：

![oldfiles.gif](https://github.com/lyb200/lua_nvim/blob/main/pic/oldfiles.gif)

如果你没有阅读过上一篇 [文章](https://github.com/lyb200/lua_nvim/blob/main/%E5%A6%82%E4%BD%95%E4%BD%BF%E7%94%A8%20lua%20%E7%BC%96%E5%86%99%20neovim%20%E6%8F%92%E4%BB%B6.md) ，我强烈建议你去读一下，因为这篇文章是在上一篇文件的基础上进行了扩展，并在相互比较中充满了新的知识。

## 插件窗口

好的，那么我们应该开始编写一个函数来创建第一个窗口，以显示打开过的文件列表。首先，我们将在脚本的主作用域中声明三个变量：`buf` 、`win` 和 `start_win` 。其中的变量 `win` 保存导航窗口和缓冲区的引用，`start_win` 将记住打开导航的位置。通过插件中的函数，我们会经常使用它们。

```lua
-- 这是主启动函数。现在我们只会在这里创建导航窗口。
local function oldfiles()
  create_win()
end

local function create_win()
  -- 我们在最右边打开一个新的垂直窗口
  vim.api.nvim_command('botright vnew')
  -- 我们把导航窗口句柄保存到变量 win 中
  win = vim.api.nvim_get_current_win()
  -- ... 然后把它的缓冲区句柄保存在变量 buf 中。
  buf = vim.api.nvim_get_current_buf()

  -- 我们应该对缓冲区命名。因为 vim 中所有的缓冲区必须有唯一的名称。最简单的解决方法
  -- 可能是给它添加缓冲区的句柄，因为句柄本身是唯一的、且仅仅是一个数字。
  vim.api.nvim_buf_set_name(buf, 'Oldfiles #' .. buf)

  -- 现在我们为缓冲区设置一些选项。
  -- nofile 选项会阻止把缓冲区标记为已修改，因此我们从来不会得到已经修改未保存的警告。
  -- 另外一些插件处理nofile缓冲区是不同的。例如 coc.nvim 不会对这些触发补全动作。
  vim.api.nvim_buf_set_option(buf, 'buftype', 'nofile')
  -- 我们设置这个缓冲区不需要交换文件。
  vim.api.nvim_buf_set_option(buf, 'swapfile', false)
  -- 并且我们喜欢这个缓冲区在隐藏时被销毁。
  vim.api.nvim_buf_set_option(buf, 'bufhidden', 'wipe')
  -- 设置自定义的文件类型是不必要的，但这是很好的实践。
  -- 这允许用户在文件类型上创建他们的自动命令或颜色主题，且防止与其他插件发生冲突。
  vim.api.nvim_buf_set_option(buf, 'filetype', 'nvim-oldfiles')

  -- 为了更好地用户体验，我们会关闭自动换行和打开当前行高亮显示。
  vim.api.nvim_win_set_option(win, 'wrap', false)
  vim.api.nvim_win_set_option(win, 'cursorline', true)

  -- 最后我们为导航设置映射。
  set_mappings()
end
```

## 绘图函数

好了，我们已经有了一个窗口，现在需要一些内容在里面显示。我们将使用 `vim oldfiles` 这个特殊变量，它保存着先前打开过的文件路径。在脚本中，我们将从中得到尽可能多的条目，因为我们可以采取不滚动显示，当然，你可以获取你想要的数量。我们把该函数命名为 `redraw` ，因为它能用于更新导航内容。文件路径可能会很长，因此我们将尝试把它们转换成当前工作目录的相对路径。

```lua
local function redraw()
  -- 首先，我们允许缓冲区可修改。最后我们会阻止这个选项。
  wim.api.nvim_buf_set_option(buf, 'modifiable', true)

  -- 得到窗口的高度
  local items_count = vim.api.nvim_win_get_height(win) - 1
  -- 译者：用于保存结果的数组
  local list = {}

  -- 如果你使用每日构建版本neovim，你可以像这样得到打开过的文件列表
  local oldfiles = vim.v.oldfiles
  -- 而在稳定版本只能这样编码
  local oldfiles = vim.api.nvim_get_vvar('oldfiles')

  -- 现在我们从打开过的文件列表中填充 x 个最近的条目
  for i = #oldfiles, #oldfiles - items_count, -1, do
    -- 我们使用内置的 `vim` 函数 `fnamemodify` 来得到相对路径
    -- 在每日构建版本中，我们可以像这样调用 vim 函数
    local path = vim.fn.fnamemodify(oldfiles[i], ':.')
    -- 而在稳定版本中，这样：
    local path = vim.api.nvim_call_function('fnamemodify', {oldfiles[i], ':.'})

    -- 由于我们采用了逆序迭代，因此应该在 list 的末端插入这些条目以保持原有的次序
    table.insert(list, #list + 1, path)
  end

  -- 我们把结果填充到缓冲区中
  vim.api.nvim_buf_set_lines(buf, 0, -1, false, list)
  -- 最后关闭缓冲区的编辑功能
  vim.api.nvim_buf_set_lines(buf, 'modifiable', false)
end
```

现在可以更新函数了。我们也会加入一些代码以防止打开多个导航窗口。为了这个目的，我们可以使用 `nvim_win_is_valid` 函数，它会检查插件窗口是否已经存在。

```lua
local function oldfiles()
  if win and vim.api.nvim_win_is_valid(win) then
    vim.api.nvim_set_current_win(win)
  else
    create_win()
  end

  redraw()
end
```

## 打开文件

现在我们能看到打开过的文件列表，但是如果也能打开它们，那就更好了。我们允许用户以 5 中不同的方式打开文件！即在新的 tab 中、垂直或水平分隔窗口中、当前窗口中和预览窗口中，而此时焦点仍然在导航中。

让我们从在当前窗口中打开文件开始。我们应该准备两个场景：

1. 在用户已经打开的导航窗口中打开文件。
2. 当我们为了打开文件而创建一个新的窗口时，关闭启动窗口。

```lua
local function open()
  -- 当用户点击回车时，从当前行中得到文件路径
  local path = vim.api.nvim_get_current_line()

  -- 如果其他窗口存在
  if vim.api.nvim_win_is_valid(start_win) then
    -- 我们把焦点移动到这个窗口
    vim.api.nvim_set_current_win(start_win)
    -- 然后编辑选择的文件
    vim.api.nvim_command('edit '.. path)
  else
    -- 如果没有起始窗口，我们在左侧创建一个新的窗口
    vim.api.nvim_command('leftabove '.. path)
    -- 且把它设置成新的起始窗口
    start_win = vim.api.nvim_get_current_win()
  end
end

-- 在打开所需的文件后，用户不需要导航了，因此我们应该创建函数来关闭它。
local function close()
  if win and wim.api.nvim_win_is_valid(win) then
    vim.api.nvim_win_close(win, true)
  end
end

-- 好了。现在我们准备创建两个一等打开函数

local function open_and_close()
  open()      -- 打开新文件
  close()     -- 关闭导航
end

local function preview()
  open()      -- 打开新的文件
  -- 但是焦点在预览窗口中，而不是关闭导航
  -- 我们把焦点移回到原窗口中
  vim.api.nvim_set_current_win(win)
end
```

```lua
-- 为了打开分隔窗口，我们需要一个函数
local function split(axis)
  local path = vim.api.nvim_get_current_line()

  -- 我们仍然需要处理两种场景
  if vim.api.nvim_win_is_valid(start_win) then
    vim.api.nvim_set_current_win(start_win)
    -- 如果想要垂直分隔窗口，传递 v 给 axis 参数，否则，不需要传递字符串
    vim.api.nvim_command(axis .. 'split ' .. path)
  else
    -- 如果没有起始窗口，我们在左侧创建一个
    vim.api.nvim_command('leftabove ' .. axis .. 'split ' .. path)
    -- 但是，在这个案例中，不需要设置新的起始窗口，因为新建分隔窗口时终是关闭导航
  end

  close()
end
```

最后，最简单的是在新的 tab 中打开。

```lua
local function open_in_tab()
  local path = vim.api.nvim_get_current_line()

  vim.api.nvim_command('tabnew ' .. path)
  close()
end
```

为了让一切正常工作，我们需要添加键映射，导出所有公共函数，并增加一个触发导航的命令。

```lua
local function set_mappings()
  local mappings = {
    q = 'close()',
    ['<cr>'] = 'open_and_close()',
    v = 'split("v")',
    s = 'split("")',
    p = 'preview()',
    t = 'open_in_tab()'
  }

  for k, v in pairs(mappings) do
    -- 假设脚本放在 `lua/nvim-oldfile.lua` 文件中
    vim.api.nvim_buf_set_mapping(buf, 'n', k, ':lua require"nvim-oldfile".'..v..'<cr>', {
      nowait = true, noremap = true, silent = true
    })
  end
end

-- 在文件结尾
return {
  oldfiles = oldfiles,
  close = close,
  open_and_close = open_and_close,
  preview = preview,
  open_in_tab = open_in_tab,
  split = split
}
```

```lua
command! Oldfiles lua require'nvim-oldfile'.oldfiles()
```

就是这样！开心点，并做了很多事件！

## 整个插件

```lua
local buf, win, start_win

local function open()
  local path = vim.api.nvim_get_current_line()

  if vim.api.nvim_win_is_valid(start_win) then
    vim.api.nvim_set_current_win(start_win)
    vim.api.nvim_command('edit ' .. path)
  else
    vim.api.nvim_command('leftabove vsplit ' .. path)
    start_win = vim.api.nvim_get_current_win()
  end
end

local function close()
  if win and vim.api.nvim_win_is_valid(win) then
    vim.api.nvim_win_close(win, true)
  end
end

local function open_and_close()
  open()
  close()
end

local function preview()
  open()
  vim.api.nvim_set_current_win(win)
end

local function split(axis)
  local path = vim.api.nvim_get_current_line()

  if vim.api.nvim_win_is_valid(start_win) then
    vim.api.nvim_set_current_win(start_win)
    vim.api.nvim_command(axis ..'split ' .. path)
  else
    vim.api.nvim_command('leftabove ' .. axis..'split ' .. path)
  end

  close()
end

local function open_in_tab()
  local path = vim.api.nvim_get_current_line()

  vim.api.nvim_command('tabnew ' .. path)
  close()
end


local function redraw()
  vim.api.nvim_buf_set_option(buf, 'modifiable', true)
  local items_count =  vim.api.nvim_win_get_height(win) - 1
  local list = {}
  local oldfiles = vim.api.nvim_get_vvar('oldfiles')

  for i = #oldfiles, #oldfiles - items_count, -1 do
    pcall(function()
      local path = vim.api.nvim_call_function('fnamemodify', {oldfiles[i], ':.'})
      table.insert(list, #list + 1, path)
    end)
  end

  vim.api.nvim_buf_set_lines(buf, 0, -1, false, list)
  vim.api.nvim_buf_set_option(buf, 'modifiable', false)
end

local function set_mappings()
  local mappings = {
    q = 'close()',
    ['<cr>'] = 'open_and_close()',
    v = 'split("v")',
    s = 'split("")',
    p = 'preview()',
    t = 'open_in_tab()'
  }

  for k,v in pairs(mappings) do
    vim.api.nvim_buf_set_keymap(buf, 'n', k, ':lua require"nvim-oldfile".'..v..'<cr>', {
        nowait = true, noremap = true, silent = true
      })
  end
end

local function create_win()
  start_win = vim.api.nvim_get_current_win()

  vim.api.nvim_command('botright vnew')
  win = vim.api.nvim_get_current_win()
  buf = vim.api.nvim_get_current_buf()

  vim.api.nvim_buf_set_name(0, 'Oldfiles #' .. buf)

  vim.api.nvim_buf_set_option(0, 'buftype', 'nofile')
  vim.api.nvim_buf_set_option(0, 'swapfile', false)
  vim.api.nvim_buf_set_option(0, 'filetype', 'nvim-oldfile')
  vim.api.nvim_buf_set_option(0, 'bufhidden', 'wipe')

  vim.api.nvim_command('setlocal nowrap')
  vim.api.nvim_command('setlocal cursorline')

  set_mappings()
end

local function oldfiles()
  if win and vim.api.nvim_win_is_valid(win) then
    vim.api.nvim_set_current_win(win)
  else
    create_win()
  end

  redraw()
end

return {
  oldfiles = oldfiles,
  close = close,
  open_and_close = open_and_close,
  preview = preview,
  open_in_tab = open_in_tab,
  split = split
}
```

---

原文件网址：[How to make UI for neovim plugins in Lua ](https://dev.to/2nit/how-to-make-ui-for-neovim-plugins-in-lua-3b6e)
