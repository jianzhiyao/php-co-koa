## 改写return

return其实可以被替换为单参数且永远不返回的函数，将return解糖进行CPS变换，改写为函数参数continuation，将迭代result作为回调参数返回，为引入异步迭代做准备。

```php
<?php

final class AsyncTask
{
    public $continuation;

    public function begin(callable $continuation)
    {
        $this->continuation = $continuation;
        $this->next();
    }

    public function next($result = null)
    {
        $value = $this->gen->send($result);

        if ($this->gen->valid()) {
            if ($value instanceof \Generator) {
                // 父任务next方法是子任务的延续，
                // 子任务迭代完成后继续完成父任务迭代
                //PHP7.0语法：[$this,"next"]()等价与$this->next()
                //7.0以前可能需要使用call_user_func
                $continuation = [$this, "next"];
                (new self($value))->begin($continuation);
            } else {
                $this->next($value);
            }

        } else {
            $cc = $this->continuation;
            $cc($result);
        }
    }
}
```

```php
<?php
function newGen()
{
    $r1 = (yield newSubGen());
    $r2 = (yield 2);
    echo $r1, $r2;
    yield 3;
}
$task = new AsyncTask(newGen());

$trace = function($r) { echo $r; };
$task->begin($trace); // output: 123

```
