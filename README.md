# manosaba 开发指南

注意，本文件可在地图数据包根目录中的docs文件夹内找到，为制作组成员使用的指南文件，供有二创需求的玩家参考<br>
使用vscode打开会有更好的阅读体验

## 数据包架构简述
1. 游戏主框架
    - manosaba:framework/scoreboard/scoreboard_init 为游戏重置时调用的函数，请将你命名空间中的 **reset** 函数写入scoreboard_init中以确保重置正常

    - 每个独立的命名空间中都有一个 **scoreboard_init** (或score_init) 函数用于创建该命名空间中需要用到的积分项<br>
      同时，命名空间中也有一个 **xxx_reset** 函数用于初始化命名空间中用到的积分项、标签，结束schedule调用等操作  -*总之就是让它恢复成游戏开始前的样子*

    - 数据包开发**极度重视运行性能**，非必要尽量不使用 **每tick常驻运行** 的函数，所有具循环需求的功能均使用自调用结构进行实现<br>
        \#自调用结构: *start* -> *progress* -(终止条件,如果满足)-> schedule function *progress* \<delay\> -> ...<br>
      同时，schedule所设的 *delay* 值尽量设为 *不影响功能为前提的最大时间间隔* <br>
      **由于本地图展示实体较多，使用 @e 选择器时必须加type参数进行限制，同时尽可能避免高频运行 @e 选择器**

    - manosaba:debug/reload 可强制调用游戏重置 *调用 manosaba:framework/scoreboard/scoreboard_init*

    - **请简单注释你的代码**

    - 地图的全局数据基本储存于 manosaba:data 中，你也可以使用 *虚拟玩家* 进行存储，但请注意所有计算需要的常量必须在数据包中**声明并赋值**<br>
      即要求所有积分项依赖的函数在 scoreboard players reset * //*(清理数据缓存)* -> function manosaba:framework/scoreboard/scoreboard_init  //*(全局重置)* 后能够正常运行

    - 游戏进程常用状态积分榜:
      - 全局数据 //*储存于 manosaba:data 中* <br>
        Gaming <bool> 0 -游戏未开始  1 -游戏已开始<br>
        judging <bool> 0 -暂未发现尸体  1 -发现尸体后 *包括调查阶段与审判庭全程*<br>
        surveyT <dummy:time> >0 时:调查阶段正在进行

      - 玩家数据 //*储存于玩家实体*<br>
        Playing <bool> 0 -不在游戏中  1 -正在游戏<br>
        ready <bool> 0 -已准备  1 -未准备<br>
        dead <bool> 0 -存活  1 -死亡<br>
        deadtest <DeathCount> -死亡检测器，变为1 -> *通过所有 dead_condition 条件* -> 触发 *dead_effect* <br>
        &emsp;&emsp;&emsp;                          //*(函数位置均在manosaba:framework/dead)*<br>
        num <dummy> range:[1..20] -玩家序号<br>
        &emsp;  num有对应的team，每个玩家都会分配到自己序号对应的队伍中，队伍名：num%N (%N∈[1..20])<br>
        cd1 <dummy:time> -通常技CD值<br>
        cd2 <dummy:time> -解放技CD值<br>
        MP <dummy> -魔力值<br>
        witchProg <dummy> -玩家魔女化进度存储值<br>
        dwitchProg <dummy> -玩家魔女化进度变化量 <br>
        &emsp;  >0时，玩家将会在状态栏看到意志力下降的效果，并使witchProg增加相应的数值<br>
        &emsp;  <0时，玩家将会在状态栏看到意志力上升的效果，并使witchProg减少相应的数值<br>
        &emsp;  *该效果由 manosaba:framework/players/always_title/wcp_operate(游戏开始时自动运行) 自动操作*<br>
        &emsp;  **想对玩家魔女化进度操作请务必通过修改 dwitchProg 值 (add/remove) 来实现，否则玩家将无法收到意志力变化提示**<br>
        MagicSort <dummy> -玩家所持有的魔法种类<br>
        HP <health> -玩家当前的实际生命值<br>
        HPmax <dummy:read-only> -玩家的最大生命值 //*实时计算* **只读，请勿修改**<br>

2. 常见地图事件
    - 玩家死亡：manosaba:framework/dead/dead_effect(玩家死亡时调用) -生成尸体等效果<br>
        调用方法 / 条件：function并传入玩家数据 *(with entity xx)* / 游戏状态玩家 *deadtest = 1*

    - 任务：tasks<br>
        调用方法 / 条件：见tasks中md文件

    - 审判庭进程：manosaba:framework/tribunal<br>
        调用方法 / 条件：tribunal_start / 尸体发现后调查阶段结束

    - 游戏结束检测<br>
        调用方法 / 条件：manosaba:game/game_end/game_end_detect / **每次玩家死亡** *dead_effect 中* 或 **审判庭结束时**

    - 尸检模块<br>
        调用方法 / 条件：manosaba:framework/corpse/corpse_check / 放大镜调查尸体
     
3. 卫星模块
    - 道具模块 (props)
    - 魔法模块 (magics)
    - 任务模块 (tasks)
    - 饰品模块 (plugins)
    - 调查模块 (manosaba:framework/survey)
    - 线索模块 (clues)

    各个模块内会有对应的Markdown文件简述模块逻辑

## 功能性API

### 全局事件记录模块
  1. 全局复盘信息<br>
    位置：manosaba:game/logs<br>
    作用：记录特殊事件，在游戏结束后按时间顺序列出<br>
    调用方式：见 manosaba:game/logs/README.md<br>
      [将需要记录的文本组件存入game:logs(storage)的msg键中，调用对应目录中log_record函数 (无需传参)]

### 伤害模块
  1. 伤害效果<br>
  &emsp; 位置:damage:damages<br>
  &emsp; 作用：多种不同伤害类型所造成的异常状态<br>
  &emsp; 调用方式：玩家受到一定特定类型的伤害后自动触发

  2. 伤害值记录<br>
  &emsp; 位置:damage:damages --所用积分榜: **xxx_damage** *xxx为伤害类型*<br>
  &emsp; 作用：记录玩家受到了多少对应的伤害值<br>
  &emsp; 调用方式：同积分榜

  3. 最新伤害状态记录<br>
  &emsp; 位置:damage:damages --所用积分榜: **xxx_stat** *xxx为伤害类型*<br>
  &emsp; 作用：记录玩家最新的受伤伤害类型与伤害值（非最新的stat值将被设为0）<br>
  &emsp; 调用方式：同积分榜

  4. 利器攻击流血效果<br>
  &emsp; 位置:damage:damages/player_damage/playerdmg_save<br>
  &emsp; 作用：在玩家受到利器伤害时根据伤害值进行流血程度判定<br>
  &emsp; 调用方式：受到 *custom_data={sharp:1b}* 物品攻击时<br>

  5. 钝器攻击效果<br>
  &emsp; 位置:damage:damages/player_damage/playerdmg_save<br>
  &emsp; 作用:在玩家受到钝器伤害时通过本次伤害值加权概率进行判定钝伤效果<br>
  &emsp; 调用方式:受到 *custom_data={blunt:1b}* 物品攻击时

### 线索模块
  1. 通用线索生成方法<br>
  &emsp; 位置：clues:api/broken_item<br>
  &emsp; 作用：分为summon/replace两种方法，可**生成线索道具掉落物/替换线索道具到玩家主副手**<br>
  &emsp; 调用方式：见函数内调用范例
    
  2. 物品损坏检测<br>
  &emsp; 位置：manosaba/tags/function<br>
  &emsp; 作用：自定义道具损坏检测条件（如检测主手对应道具damage=0）,与Inventory组件共享监听器<br>
  &emsp; 调用方式：在命名空间function目录下创建break_test.mcfunction，并在manosaba/tags/function/break_test.json中加上你创建的break_test
  &emsp;&emsp;  **同时，可损坏物品统一使用烈焰棒 (blaze_rod) 作为源物品**

  3. 利器溅血<br>
  &emsp; 位置：clues:api/blood<br>
  &emsp; 作用：受到自定义标签 *{sharp:1b}* 物品攻击的玩家产生溅血效果<br>
  &emsp; 调用方式：custom_data={sharp:1b} ，需要全匹配，有多个custom_data时需要额外写对应标签的进度（可复制原来的进度加上多的自定义标签）
  
