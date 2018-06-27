---
layout: post
title: "小爱技能开发大赛-抽签技能开发分享"
date: 2018-06-27 23:17:01 +0800
comments: true
categories: 2018-06
tags: [android,小爱同学]
---
“抽签”技能的开发流程基本沿袭官方给出的[Hello World](https://xiaoai.mi.com/documents/Home?type=/api/doc/render_markdown/SkillAccess/SkillDocument/HelloWorld)的开发方式。

首先明确技能的功能，一般而言，抽签包括两个步骤，设置抽签人数，依次开始抽签。<!--more-->此外还需要支持查看已经抽签的结果。再看应用场景，因为技能的对象面向于智能音箱，而音箱的人机交互方式只有通过语音进行，所以对于给用户的交互过程既要保证功能的准确传达，又要保证其简洁与稳定。

针对上述要求，小爱开发平台使用意图与词表的交互模型，意图基于词表，平台内置了常见的词表，如颜色、数字、国家等。本技能为了追求开发准确与高效，未自定义词表，使用内置词表完成意图构建。

本技能的意图针对其功能划分为三部分，分别是抽签人数的设置，签号的抽取，查看抽签结果，其中抽签人数的设置需要使用数字词表。构建的意图如下。

{% img /images/xiaoai-table.png%}

接下来小爱平台会根据用户输入信息整合成服务器-客户端常见的`JSON`形式的`request`发向服务器，接下来就是开发者要完成的主体部分了。
```python
from xiaoai import *
import random

def outputJson(toSpeakText, is_session_end, attribute=None, openMic=True):
    xiaoAIResponse=XiaoAIResponse(to_speak=XiaoAIToSpeak(type_=0, text=toSpeakText), open_mic=openMic)
    response = xiaoai_response(XiaoAIOpenResponse(version="1.0",is_session_end=is_session_end,response=xiaoAIResponse,session_attributes=attribute))
    return response

def main(event):
    req = xiaoai_request(event)
    sum_person = 0
    selected_num = 0
    rand =0
    count = 0
    dictionary = {}
    attribute = {"sum_person":0}
    if req.request.type == 0:
        attribute['sum_person'] = 200
        attribute['dictionary'] = dictionary
        attribute["selected_num"] = 0
        return outputJson("欢迎进入抽签小工具，请先向小爱说明抽签人数，如 一共10个人", False, attribute)
    elif req.request.type == 1:
        if ((not hasattr(req.request, "slot_info")) or (not hasattr(req.request.slot_info, "intent_name"))):
            return outputJson("抱歉，我还没有听懂。请先说明抽签人数或者直接对小爱说抽签。", False, None)
        else:
            if req.request.slot_info.intent_name == 'random_sum':
                slotsList = req.request.slot_info.slots
                sum_person = [item for item in slotsList if item['name'] == 'random_sum'][0]['value']
                if int(sum_person) > 200:
                    sum_person = 200
                elif int(sum_person) < 1:
                    sum_person = 1
                attribute["sum_person"] = int(sum_person)
                attribute["selected_num"] = int(selected_num)
                attribute['dictionary'] = dictionary
                return outputJson("已设置参与人数" + str(sum_person) + "人，请大家依次对小爱说 抽签 抽取自己的签号", False,attribute)
            elif req.request.slot_info.intent_name == 'random_select':
                if hasattr(req.session, "attributes"):
                    sum_person = int(req.session.attributes["sum_person"])
                    selected_num = int(req.session.attributes["selected_num"])
                    dictionary = req.session.attributes["dictionary"]
                    attribute["sum_person"] = int(sum_person)

                if selected_num == sum_person:
                    attribute['dictionary'] = dictionary
                    attribute["sum_person"] = selected_num
                    attribute["selected_num"] = selected_num
                    return outputJson("签号都被抽完了，您可以向小爱说 所有人 来查看抽签信息或者重新设置人数哦", False,attribute)
                else:
                    rand = random.randint(1,sum_person)
                    print("dictionary: "+ str(dictionary))
                    #略
                    dictionary.setdefault(str(rand),1)
                attribute['dictionary'] = dictionary
                attribute["selected_num"] = int(selected_num+1)
                #selected_num = selected_num+1
                return outputJson("签号："+str(rand)+"，还剩"+str(sum_person-selected_num-1)+"个签位", False, attribute) 
            elif req.request.slot_info.intent_name == 'all_selected_info':
                if hasattr(req.session, "attributes"):
                    sum_person = int(req.session.attributes["sum_person"])
                    selected_num = int(req.session.attributes["selected_num"])
                    dictionary = req.session.attributes["dictionary"]
                    attribute = req.session.attributes
                
                str_output = "已经选择的签号依次为："
                j = 1
                for i in dictionary.keys():
                    str_output += str(j)+"号签:"+ str(i) +", "
                    j+=1
                str_output +="还剩"+ str(sum_person-selected_num)+"个签位未选择。"
                print(str_output)
                return outputJson(str_output, False, attribute)
            else:
                return outputJson("抱歉，我没有听懂呢", False,None)
    else:
        return outputJson("感谢使用抽签小工具，下次再见哦~", True, None, False)
```

直接看代码吧，可以根据中文看出分发的流程，`intent_name`就是上文表格的意图名称，接下来的逻辑并不是很复杂，这里对应用场景做了一些限制，如最大（默认）抽签人数为200人，可以直接跳过设置人数这个环节，这些都是为了减少用户不必要的往复交互。

由于每一次请求都是独立的，所以抽签人数和抽签结果的储存是必须要考虑的。这里采用了小米的开放云，所以平台也限制了一些其他的实现方案，在请教了小爱工程师之后采用小爱`request`中自带的`session`字段保存信息，由于数据量不大所以不会形成太重数据传输压力，`python`的小爱包是不开源的，可以到[这个链接]( https://xiaoai.mi.com/documents/Home?type=/api/doc/render_markdown/SkillAccess/SkillDocument/CustomSkills/CustomSkillsMain)查看字段定义，刚开始都不知道`JSON`该怎么传输，后来发现只要用默认的`dictionary`就可以了。
