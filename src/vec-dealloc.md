% Освобождение

Следующее, что мы должны сделать - это реализовать Drop так, чтобы у нас не 
происходила широкомасштабная утечка кучи ресурсов. Самым простым способом будет вызывать
`pop` до тех пор пока он не вернет None, и затем освободить наш буфер. Помните,
что вызов `pop` не нужен, если `T: !Drop`. В теории мы можем спросить у Rust, 
является ли `T` `needs_drop` и избежать вызова `pop`. Однако, на практике, 
LLVM *действительно* хорош в удалении такого простого независимого кода без 
побочных эффектов, поэтому я бы не стал беспокоится, если только вы не заметите, что 
он не удален (в этом случае он будет удален).

Мы не должны вызывать `heap::deallocate`, если `self.cap == 0`, так как в этом
случае мы еще на самом деле не выделили память.


```rust,ignore
impl<T> Drop for Vec<T> {
    fn drop(&mut self) {
        if self.cap != 0 {
            while let Some(_) = self.pop() { }

            let align = mem::align_of::<T>();
            let elem_size = mem::size_of::<T>();
            let num_bytes = elem_size * self.cap;
            unsafe {
                heap::deallocate(*self.ptr as *mut _, num_bytes, align);
            }
        }
    }
}
```