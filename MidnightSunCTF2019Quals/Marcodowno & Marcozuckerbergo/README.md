# Marcodowno & Marcozuckerbergo
#### Author: Karol Baryła aka mmk1
In those 2 challs we are given sites which takes parameter from URL and then interpret and render it: in Marcodowno as Markdown, in Marcozuckerbergo as graph from mermaid.js library. Our task is to create and URL which will make alert(1) pop upoin visiting the site.

## Marcodowno
Whole Markdown interpreter is interpreted as simple chain of replaces on original parameter. The one that intrests us is `.replace(/!\[([^\]]+)\]\((https?:\/\/[a-zA-Z0-9./?#]+)\)/g, '<img src="$2" alt="$1"/>')`.
Contents of alt parameter are unrestricted, so we can do something like `![aaa" onerror="alert(1)](https://google.com)` which is turned into `<img src="https://google.com" alt="aaa" onerror="alert(1)">` and solves the task.

## Marcozuckerbergo
Mermaid lets us put images into nodes ([https://github.com/knsv/mermaid/issues/548]())
We can try modifying line from answer to something like 
`Dir((<img src='' onerror='alert(1)' width='40' />))` but the parser doesn't like ( and ). So we can use the other syntax for calling functions, and end up with 
```javascript
graph LR;
    A-->B;
    Dir((<img src='' onerror='alert`1`' width='40' />))
```
which solves the task
