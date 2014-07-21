---
layout: post
title: "记录ZoneCenter的一个bug"
categories: lua
---

上周六外网环境上发现了一个bug导致用户无法登陆。经过查明是助战模块导致的问题。最后发现是一个```whilr true do```没有退出，伪代码如下：  

``` lua
 while true do
    if CountTB(npcSet) - 1 == 1 then
        break;
    end

    local rand = math.random(1, CountTB(npcSet) - 1);
    local npc = npcSet["Npc" .. rand];
    if npc then
        table.insert(npcData, npc);
        npcSet["Npc" .. rand] = nil;
        break;
    end
end
```

```npcSet```的大小只会 ```<= 11```，在```npcSet```中随机一个项，然后将随机出来的项拷贝到另外一个table中，然后置空原来的项，最后跳出```while true do```.理论上来说，这样的做法是不可能出现这种情况的，问题就是```npcSet```是一个```hash table```，并非一个数组，这样导致的一个问题就是，当随机到了一个项时，```npcSet["Npc" .. rand] = nil```把这个项从```hash table```中干掉了，但是```CountTB```计算的是哈希表中的项个数，这样导致的问题就是在达到一定数目之后，即使表中已经随机到的项不存在了，但是```math.random```依旧会随机到已经为```nil```的值，当```npcSet```的数目足够小的时候，就会陷入死循环。

解决做法：将有可能使用的项单独拿出来构造一个数组，不使用```hash table```，通过数组随机取值：  

``` lua
function ReContructNpcData(npcSet)
    local npcData = {};
    for k, v in pairs(npcSet) do
        if k ~= "xxxxx" then
            table.insert(npcData, v);
        end
    end

    return npcData;
end

-- reconstruct a 'npc table', ensure item doesn't repeat.
npcSet = ReContructNpcData(npcSet);
for j = 1, times do
    local select = 0;
    while true do
        local rand = 0;

		-- ensure this 'while tue do' excution five times. 
        if select > 5 then
            break;
        end

        select = select + 1;
		
		-- extra ensure.
        if CountTB(npcSet) == 0 then
			break;
        end

        rand = math.random(1, CountTB(npcSet));
        local npc = npcSet[rand];
        if npc then
            table.insert(npcData, npc);

			-- remove item by refenerce 'rand'
            table.remove(npcSet, rand);
            break;
        end
    end
end
```

在探索模块中涉及到很多随机的问题，我是构建的数组进行随机，在助战模块中，本来计划使用数组随机，发现可以在```hash table```中完成此操作，但是没有正确的预估到在```hash table```随机可能出现的问题，导致了周六出现的bug，记录下这个bug，作为一个教训留存。

