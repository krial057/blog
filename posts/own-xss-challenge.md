> :Hero src=https://images.unsplash.com/photo-1486312338219-ce68d2c6f44d?w=1900&h=600&fit=crop,
>       mode=light,
>       target=desktop,
>       leak=156px

> :Hero src=https://images.unsplash.com/photo-1486312338219-ce68d2c6f44d?w=1200&h=600&fit=crop,
>       mode=light,
>       target=mobile,
>       leak=96px

> :Hero src=https://images.unsplash.com/photo-1508780709619-79562169bc64?w=1900&h=600&fit=crop,
>       mode=dark,
>       target=desktop,
>       leak=156px

> :Hero src=https://images.unsplash.com/photo-1508780709619-79562169bc64?w=1200&h=600&fit=crop,
>       mode=dark,
>       target=mobile,
>       leak=96px

> :Title shadow=0 0 8px black, color=white
>
> My first XSS challenge - Writeup

> :Author src=github

<br>

So this is the content of this sample blog post. For a new blogpost, just copy the contents of \
`posts/sample-blog-post.md` and get started quickly!

---
```php | index.php
<?php 
  $nonce = base64_encode(random_bytes(32));
  header("Content-Security-Policy: default-src 'none'; script-src 'nonce-".$nonce."'"); // --> Only allow script execution with the right nonce token
?>
<html>
    <body>
        <?php
            $todos = '';
            $todos = $todos."<div> Todo #1 </div>\n"; // --> A constant TODO
            $todos = $todos."<div> Todo #2 </div>\n"; // --> A constant TODO
            $todos = $todos."<div> ".$_GET['todo']." </div>\n"; // --> A TODO supplied by the user over url
            $todo_list = explode("\n", $todos);
            foreach($todo_list as $todo) {
                if(strpos($todo, "script") === false) { // --> Don't allow script injections in todos
                    echo $todo."\n";
                } 
            }
        ?>

        <script nonce="<?php echo $nonce ?>">
            window.onload = () => {
                console.log("bla")
            }
        </script>
    </body>
</html>
```

> :DarkLight
> > :InDark
> >
> > _Hero image by [Kaitlyn Baker](https://unsplash.com/@kaitlynbaker) from [Unsplash](https://unsplash.com)_
>
> > :InLight
> >
> > _Hero image by [Glenn Carstens-Peters](https://unsplash.com/@glenncarstenspeters) from [Unsplash](https://unsplash.com)_