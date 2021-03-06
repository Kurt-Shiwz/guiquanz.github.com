---
layout: post
title: 遍历Lua全局环境变量
date: 2012-08-01
categories:
    - 技术
tags:
    - lua
---
## Lua全局变量

`Lua`解释器提供了很多全局变量，比如`print`等，便于程序开发。Lua提供的所有全局变量都保存在一个普通的表`_G`中。目前`Lua-5.2.1`中_G中的全局变量主要有`“字符串”`、`“函数”`及`“表”`三种。那该如何遍历这些值呢？（当然，你在会话中进行的任何变更，都将对其造成影响，除非有`local`限定）

## 如何遍历Lua提供的全局变量？

既然`_G`是一个普通的表，那么我们可以采用`for`语句对其进行简单的遍历即可。具体代码，如下：

<pre class="prettyprint linenums">
for k,v in pairs(_G) do 
  print(string.format("%s => %s\n",k,v))
end
</pre>

当然，为了遍历所有的全局变量（比如，io库支持的所有函数等），我们需要对所有`table`类变量进行遍历（由于`_G._G与_G等效`，所以以下的代码中对其进行了过滤，不重复遍历。并且，此实现仅遍历了2层）。具体代码，如下：

<pre class="prettyprint linenums">
for k,v in pairs(_G) do 
  print(string.format("%s => %s","_G." .. k,v))
  if type(v) == "table" and k ~= "_G" then
     for key in pairs(v) do
        print("  " .. k .. "." .. key) --为了便于阅读，进行了特殊格式化处理
     end
  end
end
</pre>
示例代码最终执行结果，如下：

<pre class="prettyprint linenums">
$ ./src/lua
Lua 5.2.1  Copyright (C) 1994-2012 Lua.org, PUC-Rio
> 
> for k,v in pairs(_G) do 
>>   print(string.format("%s => %s","_G." .. k,v))
>>   if type(v) == "table" and k ~= "_G" then
>>      for key in pairs(v) do
>>         print("  " .. k .. "." .. key)
>>      end
>>   end
>> end
_G.require => function: 0x10da890
_G.module => function: 0x10da820
_G.pcall => function: 0x419260
_G.type => function: 0x418830
_G.dofile => function: 0x418fd0
_G.load => function: 0x4195d0
_G.rawget => function: 0x418c30
_G.math => table: 0x10dd460
  math.frexp
  math.deg
  math.tanh
  math.huge
  math.ceil
  math.acos
  math.pi
  math.asin
  ...
>
</pre>

如果只想遍历`os`库提供的函数列表，可以按以下的方式处理：

<pre class="prettyprint linenums">
> for k,v in pairs(_G.os) do print(k) end 
time
getenv
remove
exit
date
clock
tmpname
difftime
setlocale
rename
execute
</pre>

## 比较优雅的实现：基于递归的版本

以下是一个基于递归的实现版本（为什么会对package进行特殊处理呢？（提示：死循环，可进一步优化处理））：

<pre class="prettyprint linenums">
function re_print(t,prefix)
  for k,v in pairs(t) do
    if type(v) == "string" then
      print(string.format("%s => %s", prefix .. "." .. k,v))
    else
      print(prefix .. "." .. k)
    end
    if type(v) == "table" and k ~= "_G" and k ~= "_G._G" and not v.package then
      re_print(v, "  " .. prefix .. "." .. k)
    end
  end
end
</pre>

最终运行结果，如下：

<pre class="prettyprint linenums">
> function re_print(t,prefix)
>>   for k,v in pairs(t) do
>>     if type(v) == "string" then
>>       print(string.format("%s => %s", prefix .. "." .. k,v))
>>     else
>>       print(prefix .. "." .. k)
>>     end
>>     if type(v) == "table" and k ~= "_G" and k ~= "_G._G" and not v.package then
>>       re_print(v, "  " .. prefix .. "." .. k)
>>     end
>>   end
>> end
> re_print(_G, "_G")
_G.next
_G.bit32
  _G.bit32.rrotate
  _G.bit32.btest
  _G.bit32.extract
  _G.bit32.replace
  _G.bit32.bxor
  _G.bit32.band
  _G.bit32.bnot
  _G.bit32.lrotate
  _G.bit32.arshift
  _G.bit32.rshift
  _G.bit32.lshift
  _G.bit32.bor
_G._G
_G.coroutine
  _G.coroutine.running
  _G.coroutine.wrap
  _G.coroutine.create
  _G.coroutine.resume
  _G.coroutine.status
  _G.coroutine.yield
_G.assert
_G.type
_G.print
_G.module
_G.debug
  _G.debug.gethook
  _G.debug.traceback
  _G.debug.setuservalue
  _G.debug.setupvalue
  _G.debug.getlocal
  _G.debug.setmetatable
  _G.debug.getuservalue
  _G.debug.setlocal
  _G.debug.getregistry
  _G.debug.sethook
  _G.debug.getupvalue
  _G.debug.upvalueid
  _G.debug.debug
  _G.debug.upvaluejoin
  _G.debug.getmetatable
  _G.debug.getinfo
_G.rawequal
_G._VERSION => Lua 5.2
_G.string
  _G.string.len
  _G.string.find
  _G.string.gsub
  _G.string.gmatch
  _G.string.reverse
  _G.string.byte
  _G.string.format
  _G.string.rep
  _G.string.match
  _G.string.dump
  _G.string.upper
  _G.string.sub
  _G.string.char
  _G.string.lower
_G.pcall
_G.table
  _G.table.concat
  _G.table.pack
  _G.table.insert
  _G.table.sort
  _G.table.remove
  _G.table.maxn
  _G.table.unpack
_G.unpack
_G.package
  _G.package.seeall
  _G.package.config => /
;
?
!
-

  _G.package.cpath => /usr/local/lib/lua/5.2/?.so;/usr/local/lib/lua/5.2/loadall.so;./?.so
  _G.package.searchers
    _G.package.searchers.1
    _G.package.searchers.2
    _G.package.searchers.3
    _G.package.searchers.4
  _G.package.path => /usr/local/share/lua/5.2/?.lua;/usr/local/share/lua/5.2/?/init.lua;/usr/local/lib/lua/5.2/?.lua;/usr/local/lib/lua/5.2/?/init.lua;./?.lua
  _G.package.searchpath
  _G.package.loadlib
  _G.package.preload
  _G.package.loaded
  _G.package.loaders
    _G.package.loaders.1
    _G.package.loaders.2
    _G.package.loaders.3
    _G.package.loaders.4
_G.dofile
_G.error
_G.loadstring
_G.io
  _G.io.stdin
  _G.io.tmpfile
  _G.io.stderr
  _G.io.stdout
  _G.io.input
  _G.io.type
  _G.io.flush
  _G.io.lines
  _G.io.write
  _G.io.output
  _G.io.close
  _G.io.open
  _G.io.read
  _G.io.popen
_G.collectgarbage
_G.xpcall
_G.math
  _G.math.atan2
  _G.math.pi
  _G.math.asin
  _G.math.fmod
  _G.math.log10
  _G.math.cosh
  _G.math.acos
  _G.math.sqrt
  _G.math.log
  _G.math.tan
  _G.math.deg
  _G.math.modf
  _G.math.sin
  _G.math.pow
  _G.math.atan
  _G.math.tanh
  _G.math.frexp
  _G.math.randomseed
  _G.math.sinh
  _G.math.rad
  _G.math.random
  _G.math.abs
  _G.math.min
  _G.math.max
  _G.math.ldexp
  _G.math.huge
  _G.math.floor
  _G.math.exp
  _G.math.cos
  _G.math.ceil
_G.ipairs
_G.setmetatable
_G.rawget
_G.loadfile
_G.re_print
_G.os
  _G.os.clock
  _G.os.exit
  _G.os.date
  _G.os.tmpname
  _G.os.getenv
  _G.os.time
  _G.os.rename
  _G.os.setlocale
  _G.os.execute
  _G.os.remove
  _G.os.difftime
_G.tonumber
_G.getmetatable
_G.load
_G.require
_G.pairs
_G.tostring
_G.select
_G.rawset
_G.rawlen
> 
</pre>

