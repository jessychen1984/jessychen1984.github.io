---
layout: post
title: "MiniProj之简单计算题出题器"
subtitle: 'A simple calculation question generator'
author: "jessychen"
header-style: text
tags:
  - Python
  - CS
  - MiniProj
---

最近，家有小儿初长成。曾经萌萌哒的Sam小盆友也马上要步入小学生的行列了。为了不输在起跑线上，老母亲决定每天给出100题计算题😏。手动出题太麻烦，肿么办？当然是动手写一个了！用上伟大的**Python**，分分钟搞定不在话下：

```python
'''
A simple python calculation question generator
usg: python calculator.py -d "+,-" -c 3 -l 200 -t 100
usg: python calculator.py --op_type="+,-" --op_count=3 --limit=20 --total=100
'''
import sys, getopt, random

def auto_cal_generator(limit=100, op_count=1, op_type=["+"], total=100):
    print("Here are today's %d works, good luck!" % total)
    l = len(op_type)-1
    for j in range(0, total):
        up = limit
        question = ""
        for i in range(0, op_count+1):
            num = random.randint(1,max(1,min(limit,up))) 
            if i == 0:
                question = "%s%d" % (question, num)
                up -= num
                continue
            op_i = random.randint(0,l)
            op = op_type[op_i]
            question = "%s%s%d" % (question, op, num)
            if op =="+":
                up -= num
            elif op == "-":
                up += num
            else:
                print("operator error: %s" % op)
                sys.exit(1)
        
        print("%d: %s=" % (j+1, question))

def main(argv=None):
    if argv is None:
        argv = sys.argv
    opts, _dummy = getopt.getopt( sys.argv[1:], "l:c:d:t:", ["limit=","op_count=","op_type=","total="])
    limit = 100
    op_count = 1
    op_type = ["+"]
    total = 100
    
    for opt,arg in opts:
        if opt in ["-l", "--limit"]:
            limit = int(arg)
        elif opt in ["-c", "--op_count"]:
            op_count = int(arg)
        elif opt in ["-d", "--op_type"]:
            op_type = arg.strip().split(",")
        elif opt in ["-t", "--total"]:
            total = int(arg)

    auto_cal_generator(limit, op_count, op_type, total)

if __name__ == '__main__':
    main()  
```

有了这个神器后，想看电视？想吃冰激凌？想去游乐园？先做30题解锁，用法：

```shell
#出30题200以内3个运算符的加减法计算
python calculator.py -d "+,-" -c 3 -l 200 -t 30
```

看看结果：

![auto_cal_q](/img/in-post/MiniProj/auto_cal_q.png)

当然，对于不到一年级的小豆包来说，用加减法对付已经戳戳有余了，乘除法等有空再加吧。

项目代码[点此](https://github.com/jessychen1984/MiniProj/blob/master/src/misc/calculator.py)下载

