1. 目前nvm database里面的gc没有运行是吗？

2. 之前讨论的BUG是什么？

3. 是要用transaction level的GC还是用background vacuuming的方式实现GC

4. 是要通过snapshot回收GC还是通过interval

5. 如果要用snapshot来回收那就要看snapshot的实现，interval的实现比较复杂

update:
    commit 时候加入，之前的版本
    abort  时候，把自己加进去
remove:
    commit 时候把自己加进去


## Commit

tx 10

tx 11 
head -> [5, 正无穷) -> [0, 5)
while(version != null){
    version = version.next;
}

tx 10   Head -> [10, +00)  -> [5, 10) -> [0, 5)

10 提交的时候，TID 已经递增到了 15 ，future  transaction id > 15. 
从16开始，一定能看到10的结果，

active transaction id, 最小值 - 1
只要<=15 的事务，都提交了，那么是可以回收 tx10 造成的垃圾。

## Abort 

tx 10   Head -> [10, +00)  -> [5, 10) -> [0, 5)
        head -> [5, +00) ->[0, 5)

tx abort 的时候，发现 是TID是15，等到 <= 15的事务都完成了，才能回收这个。

tid: GetNewTransactionId() in transaction.h
    应该要加一个新的函数直接可以读到tid

lvd_id: where



# 思考
access_version改成if(!snapshot.is_visible(rv) && rv.xmin <= txid) 立即返回

这种情况是否可行。假设场景，tx3.commit/abort, 读到的max_seen_tid = 5，之后tx6.read(),会不会读到已经回收的版本链。假设把所有for循环改成先修改指针，读max_seen_tid，再schedule

假如这种思考方式不行，就从snapshot.is_visible里面的所有情况看看能不能传出一个值，给上层判断

## commit

### update:
可行，因为tx6是在tx3的max_seen_tid=5之后，那么tx6能看到最新的tx3的rv，那么就不会看到tx3后面的rv

思考：update的时候直接断链合法吗？合法，有必要吗？

答案：合法的。
    1. header->[3, +inf]->[1, 3]-> ：在tx3 update之后，

### remove
1. 这种情况是不行的，因为tx6.read()一定在max_seen_tid之后，那么必然会访问rh->new，如果tx6执行比tx3快，那么就会读到已经要被schedule的rv
```
    commit():
        COMMITTED

        max_seen_tid = ReadNewTransactionId() // 5

        for each remove item:
            update rh->new = nullptr
            schedule(item, max_seen_tid)
```

2. 先改remove对应的指针，然后再max_seen_tid，最后再schedule
```
    commit():
        COMMITTED

        for each remove item:
            update rh->new = nullptr
            
        
        max_seen_tid = ReadNewTransactionId() // 5

        for each remove item:
            schedule(item, max_seen_tid)

```



## abort

### update

```
    abort():

    for each update item:
        assert(rh->new = item.rv)
        rh->new = rv.older

    max_seen_tid = ReadNewTransactionId()

    for each update item:
        schedule()


```

## gc
主要问题在于：gc_ts相关的部分要怎么处理，应该只会在free rh的时候更新。abort-update, commit-remove以及commit-update都会断链


    free rv

    if rh->new == null
        free head
        gc_ts++

问题：


header->[1, +inf]

                    tx2.begin()
tx3.begin()
tx3.update(0)

header->[3, +inf]->[1, 3]

                    tx2.remove(0) // tx2会拿到[1,3]这个rv并且加入remove_items

header->[3, +inf]->[1, 2]

tx3.commit()
                    tx2.commit()

header->[3, +inf]->**[1, 2]


！！！问题2：update abort and update

header->[1, +inf]

tx3.begin()
                        tx4.begin()
tx3.update()

header->[3, +inf]->[1, 3]

tx3.abort():
    status = ABORTED

                        tx4.update // tx4会拿到[1,3]

header->[4, inf]->[1,4]
[3, +inf]->[1,4]

问题3： commit Read

header->[1, +inf]
                            tx2.begin()
tx3.begin()
                        
tx3.update()

header->[3, +inf]->[1, 3]

tx3.commit()
                            tx2.read() // tx2应该读到什么

问题3： remove Read

header->[1, +inf]
                            tx2.begin()
tx3.begin()
                        
tx3.remove()

header->[3, 3]->[1, 3]

tx3.commit()
                            tx2.read() // tx2应该读到什么
                            

update不断链
remove的时候不断链

在access version里面加入判断rh是否被自己的snapshot之前的删除了，如果是的不访问

在GC里面要加入update和remove的断链操作。update_abort还是单独处理






1. remove相关的bug:

tx3先更新了，tx4拿到了更早的rv，并且把更早的rv加入remove_item

2. update abort相关的bug：

