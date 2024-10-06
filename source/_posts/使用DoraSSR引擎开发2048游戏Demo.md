---
title: 使用DoraSSR引擎开发2048游戏Demo
date: 2024-08-11 21:32:15
tags: 
- Lua
- DoraSSR
categories: 经验分享
cover: 
index_img: ./使用DoraSSR引擎开发2048游戏Demo/logo.png
banner_img:
---

#  引言

## 介绍2048游戏

### 背景与历史

2048游戏是由意大利开发者Gabriele Cirulli在2014年创建的简易益智游戏。值得注意的是，这款游戏是在他用JavaScript自学编程时开发的，仅用了一个周末的时间。

这款游戏最初的灵感来自于一个类似的游戏——1024，而1024本身也是基于另一个名为Threes的游戏。这种创新与简化使得2048在发布后迅速走红，吸引了全球玩家的注意，并在短时间内成为了一个网络现象。

### 游戏规则

2048的游戏规则非常简单，玩家在一个4x4的网格中通过上下左右方向键滑动方块。

当两个相同数字的方块相遇时，它们会合并成一个新方块，并且数值相加（例如两个“2”合并成“4”）。

每次滑动时，所有方块会向玩家指定的方向移动，且会在空白位置随机生成一个新的方块，通常为“2”或“4”。

游戏的目标是在网格中创造一个数字为2048的方块。当玩家无法再移动方块且没有生成2048方块时，游戏结束。

### 流行原因
2048之所以迅速流行，原因在于它的极简设计和易于上手的玩法。这款游戏结合了策略性和运气成分，使得玩家在短时间内沉浸其中，享受逐步逼近2048的快感。游戏的开源性也促进了它的传播，众多开发者基于原版开发了各类衍生版本和改编游戏，进一步扩大了其影响力。

## 文章目的

本文将介绍您如何基于DoraSSR环境编写2048游戏的程序，以及其中的重要功能和技术实现。

# 开发环境

## 编程语言
如果你是一名游戏开发的新手，你可能对如何选择合适的编程语言感到困惑。别担心，Dora SSR 提供了多种语言选择，让你可以根据自己的兴趣和项目需求来挑选。

Dora SSR 支持多种编程语言，目前包括 Lua、Yuescript、Teal、TypeScript、TSX、Rust 等。每种语言都有其独特的特点和优势。

Lua 是一种轻量级的脚本语言，以其语言特性精简，高效和易学而闻名。如果你是编程新手，或者喜欢简单明了的代码，Lua 是一个很好的选择。

## 开发工具
- 下载并运行[软件](https://github.com/ippclub/Dora-SSR/releases/latest)

  - 对于 Windows 用户，请确保您已安装 Visual Studio 2022 的 X86 Visual C++ 可再发行组件包（包含 MSVC 编译的程序所需运行时的 vc_redist.x86 补丁），以运行此应用程序。您可以从[微软网站](https://learn.microsoft.com/zh-cn/cpp/windows/latest-supported-vc-redist?view=msvc-170)下载。

  - 在macOS上也可以通过Homebrew进行软件安装。

    ```sh
    brew tap ippclub/dora-ssr
    brew install --cask dora-ssr
    ```

- 通过浏览器访问软件显示的服务器地址。

- 开始游戏开发。

# 程序架构设计

## 整体结构

- **模块划分**
  该程序的整体架构采用了模块化设计。它通过引入多个模块（如`Scale`、`Audio`、`Set`等）来处理不同的功能。这种设计使得代码更加清晰且易于维护。主要模块包括：
  - **主逻辑模块**：处理游戏的初始化、主要逻辑控制以及事件响应。
  - **UI模块**：负责绘制游戏界面，包括游戏板和方块。
  - **工具模块**：提供一些常用的辅助函数，如遍历表格和查找元素。
- **数据结构**
  游戏的核心数据结构是一个一维数组`data`，用于存储每个格子中的数字状态。每个数字方块在界面上对应一个由`DrawNode`绘制的方块对象，保存在`boxes`数组中。

## 关键模块与功能

- **输入处理**
  程序通过键盘输入来捕捉玩家的方向操作，使用`slot`函数来监听用户按键事件，并调用相应的逻辑处理函数进行处理。
- **游戏逻辑**
  包括方块的生成、移动、合并及重新开始游戏的逻辑。通过`gameMove()`函数来处理方块的移动，并根据不同的方向（左、右、上、下）调用合并函数`merge(newRow)`。
- **界面绘制**
  使用`DrawNode`绘制方块，通过`Label`显示方块上的数字，并将它们添加到游戏主节点`root`中。

# 核心功能实现

## 游戏初始化

- 游戏通过`init()`函数进行初始化。在初始化过程中，程序会将数据表`data`中的值全部重置为0，并在随机位置生成一个初始的数字“2”。

- 同时，`init()`函数设置了键盘输入的监听器，允许玩家通过键盘控制方块的移动。

  ```lua
  local function init()
  	for i = 1, 16, 1 do
  		empty:add(i)
  		data[i] = 0
  	end
  	
  	local rand0 = empty:random_element()
  	data[rand0] = 2
  	drawNumber()
  
  	root.keyboardEnabled = true
  	root:slot("KeyDown", function(keyName)
  		if MyMethod.findElement(Direction, keyName) ~= nil then
      		direction = keyName
  			gameMove()
  			print("direction:", keyName)
  		else print("Unknown direction:", keyName) end
  	end)
  end
  ```

## 方块的绘制

- `draw(x, y)`函数负责绘制单个方块。它使用`DrawNode`来创建方块的形状，并在其上添加一个`Label`用于显示数字。绘制时，方块会被定位到指定的坐标（`x`, `y`），并添加到主节点`root`中。

  ```lua
  local function draw(x ,y)
  	local drawNode = DrawNode()
  	drawNode.position = Vec2(x, y)
  	drawNode:drawPolygon(verts, Color(0xFF072E06), 5, Color(0xFF2C6AC7))
  	local lable = Label('sarasa-mono-sc-regular', 80)
  	lable.text = ''
  	drawNode:addChild(lable)
  	drawNode:addTo(root)
  	return drawNode
  end
  ```

## 方块的移动与合并

- 当用户按下某个方向键时，程序会通过`gameMove()`函数处理方块的移动。根据方向的不同，程序会将数据分成不同的行或列，并通过`merge(newRow)`函数来合并相同的数字。

  ```lua
  local function gameMove()
  	if direction == "Left" then
  		for i = 1, 4, 1 do
  		-- i  + 0/1/2/3 * 4
  			local newRow = {}
  			for j = 0, 3, 1 do
          	if data[i + j * 4] ~= 0 then
              	table.insert(newRow, data[i + j * 4])
          	end
      	end
  		while #newRow < 4 do
          	table.insert(newRow, 0)
      	end
  		local finalRow = merge(newRow)
  		for j = 0, 3, 1 do
          	data[i + 4 * j] = finalRow[j + 1]
      	end
  	end
  	elseif direction == "Right" then
  		...
  	elseif direction == "Up" then
  		...
  	else  -- Down
  		...
  	end
  	drawNumber()
  end
  ```

- `merge(newRow)`函数负责在一行数据中将相同数字的方块合并，并返回合并后的新行数据。这个函数还确保了合并后的新行保持原有的顺序，并填充为长度为4的列表。

  ```lua
  local function merge(newRow)
  	local merged = {false, false, false, false}
  	local resultRow = {0, 0, 0, 0}
  	for j = 1, 4, 1 do
          if newRow[j] ~= 0 then
            local currentValue = newRow[j]
            local nextIndex = j + 1
            if nextIndex <= 4 and newRow[nextIndex] == currentValue and not merged[nextIndex] then
              resultRow[j] = currentValue * 2
              merged[nextIndex] = true
              merged[j] = true
              newRow[nextIndex] = 0
            else resultRow[j] = currentValue end
          end
    	end
  	local finalRow = {}
    	for j = 1, 4, 1 do
      	if resultRow[j] ~= 0 then
        		table.insert(finalRow, resultRow[j])
      	end
    	end
  	while #finalRow < 4 do
      	table.insert(finalRow, 0)
    	end
  	return finalRow
  end
  ```

## 随机生成新方块

- 在每次移动之后，程序会调用`drawNumber()`函数，在空余的位置上随机生成一个新的方块。这个新方块的初始值通常为“2”。

  ```lua
  local function drawNumber()
  	empty:clear()
  	MyMethod.foreach(data, function(index, value)
  		if (value == 0) then empty:add(index) end
  	end)
  	if empty:size() == 0 then print("over") reStartGame = true end
  	local newRand = empty:random_element()
  	data[newRand] = 2
  
  	MyMethod.foreach(data, function(index, value)
  		if value ~= 0 then
  			boxes[index].children.last.text = tostring(value)
  		else boxes[index].children.last.text = '' end
  	end)
  end
  ```

  

## 游戏结束处理

- 当没有可移动的空格时，`empty:size() == 0`的判断条件会触发游戏结束逻辑，并通过`reStartGame`变量标识游戏状态。在游戏结束后，程序会播放一个音效，并重新初始化游戏。

  ```lua
  threadLoop(function()
  	if(reStartGame == true) then
  		reStartGame = false
  		Audio:play("sfx/game_over.wav")
  		init()
  		root:perform(Scale(2, 0, 1))
  	end
  end)
  ```

  

# 优化与性能提升

## 算法优化

- **合并逻辑优化**
  `merge(newRow)`函数使用了一个布尔数组`merged`来跟踪哪些方块已经合并，避免重复合并。这个设计减少了不必要的计算，提高了算法的效率。

## 内存与速度优化

- **内存管理**
  游戏中大量使用了局部变量（如`newRow`和`resultRow`），这些局部变量在函数执行完毕后自动释放，从而节省内存。
- **减少重复操作**
  在移动和合并过程中，代码尽量减少了对`data`数组的重复遍历，这有助于提升程序的整体速度。

## 用户体验优化

- **动画效果**
  虽然代码中目前未实现具体的动画效果，但可以通过增加方块移动、合并时的过渡动画（例如使用`Scale`来缩放方块），使得游戏体验更加流畅和有趣。
- **声音反馈**
  在游戏结束时播放音效，为玩家提供了即时的反馈，提高了游戏的沉浸感。

# 结论与未来改进

## 总结

- 本文详细介绍了如何从零开始编写一个2048游戏程序，并讲解了程序的整体架构设计、核心功能实现、以及优化策略。通过模块化设计和合理的算法优化，这个程序不仅具备较高的可维护性，还能在多种平台上流畅运行。
- 在开发过程中，我们通过单元测试和集成测试确保了程序的稳定性，并在性能和用户体验方面进行了优化，使得游戏更加流畅和有趣。

## 未来改进方向

- **增加难度模式**
  可以在未来的版本中增加不同的难度模式，例如更大的网格、更高的初始数字或增加障碍物，以增强游戏的挑战性。
- **添加AI对手**
  可以考虑为游戏添加一个AI对手，让玩家与AI竞赛，看谁能先达到2048。
- **改进用户界面和动画效果**
  在未来的版本中，可以进一步改进游戏的用户界面，使之更具吸引力。同时，增加方块移动和合并时的动画效果，提高游戏的视觉体验。
- **增加排行榜功能**
  通过添加在线或本地的排行榜功能，玩家可以与朋友或全球的玩家一较高下，增加游戏的社交性和竞争性。

# 附录

## MyMethod.lua

```lua
local M = {}
function M.foreach(tbl, func)
    for key, value in pairs(tbl) do func(key, value) end
end
function M.findElement(tbl, element)
    for i, value in ipairs(tbl) do
        if value == element then return i end
    end
    return nil
end
return M
```

## Set.lua

```lua
local Set = {}
Set.__index = Set
function Set.new()
    return setmetatable({ _items = {} }, Set)
end
function Set:add(item)
    self._items[item] = true
end
function Set:remove(item)
    self._items[item] = nil
end
function Set:contains(item)
    return self._items[item] ~= nil
end
function Set:size()
    local count = 0
    for _ in pairs(self._items) do
        count = count + 1
    end
    return count
end
function Set:iterate()
    local items = {}
    for item in pairs(self._items) do
        table.insert(items, item)
    end
    return items
end
function Set:print()
	for _, item in ipairs(self:iterate()) do
    print(item)
	end
end
function Set:random_element()
    local items = self:iterate()
    if #items == 0 then
        return nil
    end
    local index = math.random(1, #items)
    return items[index]
end
function Set:clear()
    self._items = {}
end
return Set
```

## 其他

- [本文工程压缩包](https://harker.lanzouw.com/iDjBJ276m0ha)
- [Lua 5.4 中文参考手册](https://atom-l.github.io/lua5.4-manual-zh/1.html)
- [Dora SSR API 参考手册](https://dora-ssr.net/zh-Hans/docs/api/intro)

