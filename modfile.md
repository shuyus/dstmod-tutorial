

## 模组位置
本地模组有两个储存位置，一个位于``Steam\steamapps\common\Don't Starve Together\mods\``下，一个在``Steam\steamapps\workshop\content\322330\``下  
在以前只有前面一个储存位置，后来克雷的更新使得新模组(所谓的v2版本模组)都改变到了后面的储存位置
新模组的储存可能不是源文件的形式，而是形如``xxxxxxxxxxxxxx_legacy.bin``这种格式的文件，它实际上是个压缩包，用压缩软件直接解压即可获得模组源文件

> 这里提一个常识性问题：文件格式取决于文件本身的安排，不取决于后缀名；
真正地识别一个文件文件取决于文件开头的若干位固定格式码(称之为魔数 `magic number`)

## 模组结构
一般模组结构如下
```txt
anim                            动画压缩包文件
exported                        scml源文件
images                          图片tex和xml文件
scripts                         代码文件
shaders                         着色器文件
modmain.lua                     入口文件
modworldgenmain.lua             入口文件(地图生成相关)
modinfo.lua                     信息文件
```
- exported
    此文件夹内放的是spriter生成的scml相关文件，在启动DST的时候，会自动使用`Don't Starve Mod Tools`里的`autocompiler.exe`自动编译成相关的bin压缩包，置入anim文件夹中  
    这个文件夹不应该出现在发布到创意工坊后的可下载项目内 

- anim
    此文件夹内存放的是对应的动画bin压缩包，里面的bin文件可以使用ktools进行反编译生成scml文件  

- images
    此文件夹内存放的是图片文件，包括tex文件和xml文件，因为一个tex里可能不止一张贴图，xml文件用于描述tex文件内各贴图的位置信息，  
    在启动DST时，也会自动将图片文件生成对应的tex和xml文件，建议使用32位深的png  
    可以使用`TEXTool`查看生成tex文件

- scripts
    此文件夹存放主要代码,内部目录结构对应源码文件夹的结构

- shaders
    此文件夹存放着色器文件,绝大多数模组不涉及此方面的文件

- modmain.lua  
    这个文件就是模组的入口文件，每一个模组的`modmain.lua`都有独立的环境表

- modworldgenmain.lua  
    这个文件是与地图生成相关的入口文件，同样的每一个模组的`modworldgenmain.lua`都有独立的环境表且与自己的`modmain.lua`的环境表也不同
    注意在加载模组的时候，这个文件先于`modmain.lua`加载

## 模组加载与覆盖  
> 这里只谈论模组文件的覆盖，不包括直接修改饥荒源文件之类的操作  

优先级低的模组后加载，会覆盖优先级高的  
因为lua模块的加载机制，当通过require加载模块的时候，会从首至尾轮询`package.path`里的路径寻找文件  

lua加载模块是有`缓存机制`的,加载模块会把模块返回的结果缓存在`package.loaded`表里  
如果第二次require一个模块，不会真正再执行一遍被加载文件，而是直接从缓存中拿返回值  
```lua
--比如我加载了foo这个模块  
require "foo"
--那么这个时候，package.loaded表里就会有一个对应的foo键
--如果foo.lua文件有返回值，那么foo键对应的值就是返回值，如果文件没有返回值，则为true
package.loaded.foo = nil
--你可以将这个元素设置为nil，再require此文件，那么文件就会重新执行，再写入缓存
```

在`mods.lua`的`ModWrangler:LoadMods`方法里，我们可以了解到模组的一部分加载细节  
```lua
--mods.lua ModWrangler:LoadMods
for i,mod in ipairs(self.mods) do
    package.path = MODS_ROOT..mod.modname.."\\scripts\\?.lua;"..package.path  
--这里就是轮询模组，低优先级模组在表后面，但是越低优先级模组的path会被连接到package.path前面  
--模组的加载源于游戏主体逻辑之前，所以require文件的时候，会优先到低优先级模组的scripts下去找
--这就是覆盖法生效的原因
```
那么我们也可以讨论一下同名文件覆盖无效的情况  
一种是更低优先级模组也覆盖了这个文件，它先加载，所以require了他的；但是这种情况的出现往往会带来更严重的模组冲突，直接造成逻辑崩溃，追究无效不无效也没有意义了  
一种可能就是其提前require了，有些文件在添加path之前就被require了，也可能是更低优先级的模组require了  

如果你使用覆盖法试图覆盖和地图生成相关的部分文件，你会发现是无效的，在`worldgen_main.lua`文件的最顶上，你会发现`package.path`被复原了


## 环境表env
> 如果你问我啥是环境表，这个是lua语言特性的范畴，可以查一下`setfenv`和`pcall`了解
简单来说,环境表规定了你在`modmain.lua`可使用的变量和函数  

游戏的lua层面逻辑都运行于lua虚拟机，都有统一的全局表`_G`   

> lua虚拟机，以下简称`lvm`  (`lua virtual machine`)

如果你阅读日志文件，你会发现你的模组实际上都是被加载了两遍的，这两次加载发生于不同的lvm之内，你可以在modmian打印一下GLOBAL的地址，两次地址是不一致的  
一次是`FrontendLoadMod`进行加载的，你可以在`mods.lua`的`ModWrangler:FrontendLoadMod`看到此过程
一次是`LoadMod`进行加载的，你可以在`mods.lua`的`ModWrangler:LoadMod`看到此过程

在`mods.lua`的`CreateEnvironment`函数中，你可以了解到`modmain`和`modworldgenmain`的环境表中究竟有什么  

```lua
function CreateEnvironment(modname, isworldgen, isfrontend)
    -- isworldgen是区分是否是生成世界的
	local modutil = require("modutil")
	require("map/lockandkey")

    -- 可以看到里面包含一些常用的lua函数，API和部分关键的表  
	local env =
	{
		pairs = pairs,
		ipairs = ipairs,
		print = print,
		math = math,
		table = table,
		type = type,
		string = string,
		tostring = tostring,
		require = require,
		Class = Class,
		TUNING=TUNING,
		LEVELCATEGORY = LEVELCATEGORY,
        GROUND = GROUND,
		WORLD_TILES = WORLD_TILES,
        LOCKS = LOCKS,
        KEYS = KEYS,
        LEVELTYPE = LEVELTYPE,
		GLOBAL = _G,  --GLOBAL指代全局表
		modname = modname,
		MODROOT = MODS_ROOT..modname.."/",
	}

	if isworldgen == false then
		env.CHARACTERLIST = GetActiveCharacterList()
	end

	env.env = env --这一行就表明了我们在modmain和modworldgenmain中使用env直接指代环境表本身  

	-- 从这里我们可以得知，modimport是klei提供的API，modimport的起始地址就是模组的根目录本身
    -- require是lua语言的文件加载机制，require的文件最后都会在全局表缓存结果，上面我们也提到了了require的加载路径是scripts和各个模组的scripts文件夹
	env.modimport = function(modulename)
		print("modimport: "..env.MODROOT..modulename)
        if string.sub(modulename, #modulename-3,#modulename) ~= ".lua" then
            modulename = modulename..".lua"
        end
        local result = kleiloadlua(env.MODROOT..modulename)
		if result == nil then
			error("Error in modimport: "..modulename.." not found!")
		elseif type(result) == "string" then
			error("Error in modimport: "..ModInfoname(modname).." importing "..modulename.."!\n"..result)
		else
        	setfenv(result, env.env)
            result()
        end
	end

	modutil.InsertPostInitFunctions(env, isworldgen, isfrontend)

	return env
end
```

### 关于"一键GLOBAL"
```lua
GLOBAL.setmetatable(env,{__index=function(t,k) return GLOBAL.rawget(GLOBAL,k) end})
```
`env`就是当前文件的环境表，这一段代码给`env`设置了一个元表，元表的`__index`键是一个函数，这个函数就劫持了数据的查询
效果就是，如果`env`表找不到，直接在`GLOBAL`表里找,这样你就可以省去`GLOBAL`的前缀了.  

> 如果你对`元表`不熟悉，可以去了解一下


    
    