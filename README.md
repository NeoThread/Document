This document is a notebook and is noted what I learned.
If I make some mistakes in my articles, you can email to me.
I will appreciate to your helpful reply.
This is my [email](mailto:baxcky@gmail.com).

The following are syntex testing.

```html
<style>
    * {
        margin: 0;
        padding: 0;
    }

    body > div {
        font-size: 4px;
        color: #fff;
    }
</style>

<script>
    $(document).ready(function(){
        var something = 5;
        if (something == 5){
            console.log('woohoo');
        }
    });
</script>

<section>
    <p>Hello world</p>
</section>
```
```uml
@startuml

  Class Stage
  Class Timeout {
      +constructor:function(cfg)
      +timeout:function(ctx)
      +overdue:function(ctx)
      +stage: Stage
  }
  Stage <|-- Timeout

@enduml
```

When $$(a \ne 0)$$,there are two solutions to $$(ax^2 + bx + c = 0)$$ and they are $$x = {-b \pm \sqrt{b^2-4ac} \over 2a}.$$

$$(a \ne 0)$$


```mermaid
graph TD;
  A-->B;
  A-->C;
  B-->D;
  C-->D;
```

$$\frac{n!}{k!(n-k)!} = \binom{n}{k}$$
$$\frac{n!}{k!(n-k)!} = {n \choose k}$$

$$\cfrac{a+\cfrac{a+b}{c}}{c}$$

$$\begin{array}{ll}
  a & b \\
  c & d
\end{array}$$
$$\sum_{i=1}^{n} a_i$$
$$\displaystyle\sum_{i=o}^{n} I_n$$
$$\int_{0}^{\infty} e^{-x}dx$$
$$\int\limits_{0}^{\infty} e^{-x}\,dx$$
$$
	{[{-1},{99}]}
	{[{1000},{-1})}
$$
$$
\begin{smallmatrix}
a&b \\
c&d
\end{smallmatrix}
\mathfrak{ABC} 
a''
\stackrel\frown{abc}
\begin{cases} 
  1 			& \text{if } x \geq 0 \\
  0       & \text{if } x < 0
\end{cases}
$$
$$
\ddots
$$
