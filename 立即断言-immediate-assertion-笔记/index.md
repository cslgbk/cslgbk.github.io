# 立即断言 Immediate Assertion笔记


### 1.断言类型
<font size=3/>立即断言(Immediate Assertion)</font>
<font size=3/>并发断言(Concurrent Assertion)</font>
<font size=3/>递归断言(Deferred Assertion)</font>

### 2.立即断言
&bull; <font size=3/>执行不消耗时间</font>

&bull; <font size=3/>可在指定位置执行</font>

&bull; <font size=3/>可以像if一样使用</font>

### 3.立即断言用法
例如：
```
always @(posedge clk)
begin
    if(a)
    begin
        @(posedge d);
        boRC: assert (b || c) 
            $display("\n", $stime, "%m assertion pass\n");
        else
            $fatal("\n", $stime, "%m assertion failed\n");
    end
end
```
<font size=3/>boRC: assert声明标志，可通过%m打印出来 </font>

<font size=3/>else: assert如果检查失败，执行else下语句，可以把assert理解为if，assert中的内容不可综合。</font>

<font size=3/>立即断言可以在代码任何地方插入，运行到断言后立即执行，不消耗时间，不需要提供采样时钟。</font>

