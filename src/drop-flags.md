% Флаги удаления

Пример в предыдущем разделе показал интересную проблему для Rust. Мы увидели,
что можно использовать условную инициализацию, деинициализацию и
переинициализацию участков памяти абсолютно безопасно. Для типов, реализующих
Copy, это не особо важно, потому что они являются просто случайной кучкой бит.
Но для типов с деструкторами - это совсем другая история: Rust нужно знать,
вызывать ли деструктор, когда переменная присваивается или выходит из области
видимости. Откуда это можно узнать в случае условной инициализации?

Заметьте, что не все присваивания должны волноваться об этой проблеме. В
частности, присваивание через разыменование безусловно вызывает деструктор, а
присваивание в `let` безусловно не вызывает его:

```
let mut x = Box::new(0); // let создает новую переменную, поэтому деструктор не 
                         // нужно вызывать 
let y = &mut x;
*y = Box::new(1); // Deref подразумевает, что референт инициализирован, поэтому 
                  // деструктор вызывается всегда
```

Это проблема возникает только при перезаписывании ранее инициализированных
переменных или их под-полей.

Выясняется, что Rust на самом деле следит за тем, нужно ли *во время исполнения*
вызывать деструктор или нет. Когда переменная становится инициализированной или
неинициализированной, ее *флаг удаления* переключается. Когда необходимо удалить
переменную, по этому флагу оценивается нужно ли вызывать у нее деструктор.

Конечно, часто можно статически определить в любом месте программы состояние
инициализации у значения. В этом случае компилятор, теоретически, может создать
более эффективный код! Например, прямолинейный код обладает такой *семантикой
статических удалений*:

```rust
let mut x = Box::new(0); // x была не инициализирована; просто перезаписать.
let mut y = x;           // y была не инициализирована; просто перезаписать и 
                         // сделать x неинициализированной.

x = Box::new(0);         // x была не инициализирована; просто перезаписать.

y = x;                   // y была инициализирована; Вызвать деструктор y, 
                         // перезаписать ее, и сделать x неинициализированной!

                         // y выходит из области видимости; y была
                         // инициализирована; Вызвать деструктор y!

                         // x выходит из области видимости; x была не 
                         // инициализирована; ничего не делать.
```

Код с условным ветвлением, где внутри веток наблюдается похожее поведение,
обладает такой же семантикой статических удалений:

```rust
# let condition = true;
let mut x = Box::new(0);    // x была не инициализирована; просто перезаписать.
if condition {
    drop(x)                 // у x забирается владение ; сделать x 
                            // неинициализированной.
} else {
    println!("{}", x);
    drop(x)                 // у x забирается владение ; сделать x 
                            // неинициализированной.
}
x = Box::new(0);            // x была не инициализирована; просто перезаписать.
                            // x выходит из области видимости; x была 
                            // инициализирована; Вызвать деструктор x!
```

Однако такому коду *требуется* информация из времени исполнения для правильного
удаления:

```rust
# let condition = true;
let x;
if condition {
    x = Box::new(0);        // x была не инициализирована; просто перезаписать.
    println!("{}", x);
}
                            // x выходит из области видимости; x возможно была 
                            // не инициализирована; проверить флаг!
```

Конечно, в данном случае легко можно получить семантику статических удалений:

```rust
# let condition = true;
if condition {
    let x = Box::new(0);
    println!("{}", x);
}
```

Что касается Rust 1.0, флаги удаления на самом деле не-так-уж-секретно спрятаны
в невидимом поле любого типа, реализующего Drop. Rust устанавливает флаг,
переписывая старое значение новым набором бит. Очевидно, что это Не Самый
Быстрый способ, вызывающий несколько проблем при оптимизации. Он пришел еще с того
времени, когда вы могли выполнять гораздо более сложную условную инициализацию.

Сейчас идет работа по переносу флагов в кадр стека, которому они по-настоящему 
и принадлежат. К сожалению, эта работа потребует немало времени, потому что
требуется внести довольно существенные изменения в компилятор.

Независимо от этого, программы на Rust не должны волноваться о корректности
неинициализированных значений в стеке. Хотя они могут волноваться о
производительности. К счастью, Rust позволяет легко контролировать её!
Неинициализированные значения существуют, и вы никогда не попадете в беду при
работе с ними в Безопасном Rust.