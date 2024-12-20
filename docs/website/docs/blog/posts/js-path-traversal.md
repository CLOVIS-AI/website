---
date: 
  created: 2024-09-23
slug: js-path-traversal
tags:
  - JS/TS
  - InfoSec
---

# The vulnerability is by design

We should all strive to write code that is harder—if not impossible—to exploit. Sometimes, however, the vulnerability comes from places we thought we could trust.

<!-- more -->

Information security (InfoSec) has become a more and more popular topic as years go by. Companies of any sizes have had major leaks, to the point that most people in countries with Internet probably suffered from one already.

Yet, software continues to be produced at high pace, often by the lowest bidder, with no regards to data integrity or privacy. InfoSec is not part of the curriculum of computer science studies, and is considered a specialty completely distinct from the job of developers. Almost all code deployed today can be trivially exploited due to poor knowledge, risk assessment or misguided priorities.

Today's story isn't about an extremely obvious flaw. If anything, this is possibly the sneakiest flaw I've seen. Not because it's complex—it is deceptively simple—but because I would have never looked there. 

## Disclaimer

This story is a real security vulnerability that has been exploited in a real production system and used to leak real user data.

However, this story isn't a disclosure about the flaw. This article is purely about the technology itself, and doesn't mention any real people, companies or events.

## A short introduction to path traversal attacks

One of the most basic attacks against a web server is the path traversal attack. Broadly, a path traversal designates a situation in which an attacker is able to access (read or write) files outside the allowed tree.

If you are already familiar with path traversal attacks, you may skip to [the puzzle](#the-puzzle).

### Files and paths

Files, and file paths, show up _everywhere_ in computer science. While modern OSs are trying to hide them more and more, they are nonetheless always there. On UNIX-inspired systems (including Linux, thus nearly the entire web), paths are addressed as `/`-separated segments. The entire filesystem lives under the _root_, a directory called `/`. A typical web server's filesystem may look something like:
```shell
/
	boot/      # Bootloader and other tools needed to start the system
	usr/       # Installed programs
	etc/       # Configuration files
	mnt/       # Attached external filesystems
	var/
		www/   # Typical location for resources published to the web
```

For example, the configuration of CRON (a tool to run tasks scheduled at regular intervals) is placed in `/etc/crontab`. Files hosted to the web are typically put in `/var/www`.

### URLs

Let's say we wanted to access, via the web, the file `/var/www/blog/index.html`. Assuming a web server is running and configured to serve files from `/var/www`, we would do so by making a request to the following URL:
```text
https://my.website.com/blog/index.html
```

Notice how the URL is split into three parts:

- `https://` is the protocol used to communicate with the web server. Nowadays, HTTPS and its unsecure sibling, HTTP, are almost the only ones still used. Many others exist, like FTP, SCP, SMB ("Windows shared drives"), etc.
- `my.website.com` determines which physical machine should be contacted to obtain the file.
- `/blog/index.html` is the path of the file we want to obtain.

Notice how the described path is really just a regular file path. The web server concatenates it with its configured root, `/var/www`, to obtain the real path: `/var/www/blog/index.html`. It then reads the file and sends it to the user.

But, you see, simply concatenating paths isn't good enough. Paths have a few special features. One of the there is the special `..` directory.

### Our first path traversal

The `..` directory is an imaginary directory that exists in all other directories on the system, and allows to link back to the parent directory. For example, if we're in the `/home/test/bar` directory and run the command `#!sh cd ..` command, we move to the `/home/test` directory: the direct parent of where we were.

We can use this special directory within paths, to move from one place to another. For example, the path `/home/test/bar/../README.md` and the path `/home/test/README.md` are strictly equivalent: the `/..` cancels out the previous `/bar`.

We can use this information to reach places we shouldn't be able to. For example, what if we wanted to read the contents of the file `/etc/passwd`: the list of all users on the system (no, it does not contain passwords). Maybe we could later bruteforce passwords for the users we find, and maybe gain further access this way.

Well, we know that our server is hosted from `/var/www`. So, the URL to access the file we want is:
```text
https://my.website.com/../../etc/passwd
```

If the web server happened to be hosted with root rights (yes, some people do this) we could directly access the file `/etc/shadow`, which does actually contain the user's password's hashes. Or, we could extract the server's SSL certificates and pretend to be them in a way that cannot be detected even with HTTPS. You get the idea.

The root directory's `..` directory is itself, meaning that `/etc/foo` and `/var/../../../../../../../etc/foo` are the same path. This is useful because it allows us to simply spam `/..` in our path traversal attempts when we do not know how deep the root is.

Thankfully, such a major flaw is extremely rare nowadays. Web servers are usually run as their own user (which cannot access the rest of the system), and any half-decent implementation will detect such attempts and block them anyway.

Most websites aren't simple static web servers, though. The average website has quite a lot of custom code, which may or may not have been written by careful developers. Services that allow uploading files are particularly at risk of incorrectly implementing path handling and thus allowing traversals. In the age of social media, most sites allow to upload user content, even it is only a profile picture.

## The puzzle

Let's play a little game. For the next few minutes, you are an attacker. You are faced with a web server that possesses an upload endpoint. Your goal is to overwrite an arbitrary file on the system. At the very least, this will cause damage to your target. Maybe, you'll be able to edit an important file.

Through some other means, you have obtained the source code of the upload service. After removing anything unrelated, you have discovered the specific code that handles writing the uploaded file to disk.

```javascript title="JavaScript"
const path = require('node:path');

const sanitizedPath = path.basename(userPath);
const link = "/shares/downloads/:uuid:/".replace(":uuid:", sanitizedPath);
```

You control the value of the `userPath` variable. Your goal is to perform a path traversal attack: store the file _elsewhere_ than in `/shares/downloads`.

Really, try to think about it. This code is very simple, right? It's just two function calls. Both are from standard libraries: one from JS', the other from NodeJS'.

<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>

I mean, really, try to solve it.

<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>
<br>

You're done? Let's see.

## Understanding the constraints

Let's analyse the code together.

> ```javascript hl_lines="3"
> const path = require('node:path');
> 
> const sanitizedPath = path.basename(userPath);
> const link = "/shares/downloads/:uuid:/".replace(":uuid:", sanitizedPath);
> ```

We—the attacker—control the `userPath` variable. On this line, the developers use the [NodeJS `path.basename` function](https://nodejs.org/api/path.html) to obtain the name of the file, removing all intermediary directories.

This is a great idea: if we try to submit the path `foo/bar/baz.txt`, all directories will be stripped and `sanitizedPath` will contain `baz.txt`. The same happens if we try to use the `..` special directory: `../../../../../etc/passwd.txt` becomes `passwd.txt`.

At this stage, the path is completely sanitized. It is not possible (at least, to my knowledge), to create a path traversal attack there. Clearly, the developers have been careful to stop these attacks.

The next line of code constructs the final path:

> ```javascript hl_lines="4"
> const path = require('node:path');
> 
> const sanitizedPath = path.basename(userPath);
> const link = "/shares/downloads/:uuid:/".replace(":uuid:", sanitizedPath);
> ```

The developers replace a part of a template path by our sanitized path, using JavaScript's standard library [`String.prototype.replace` function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace).

So, they're good, there are no path traversals here. Sorry, this article was all for naught, we can't do anything here.

## Betrayal and the final payload

I never really did read the `String.prototype.replace` function's documentation. Not entirely. I mean, it replaces a substring by another, right? The initial substring is hardcoded, and the developers sanitized the replacement. There's nothing we can do here.

Except, there's this funny table in the documentation (reproduced from the [MDN](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/replace#specifying_a_string_as_the_replacement)):

| Pattern in replacement | Inserts                                                                                      |
|------------------------|----------------------------------------------------------------------------------------------|
| `$$`                   | Inserts a `$`.                                                                               |
| `$&`                   | Inserts the matched substring.                                                               |
| <code>$`</code>        | Inserts the portion of the string that precedes the matched substring.                       |
| `$'`                   | Inserts the portion of the string that follows the matched substring.                        |
| `$n`                   | Inserts the `n`th (1-indexed) capturing group where `n` is a positive integer less than 100. |
| `$<name>`              | Inserts the named capturing group where `Name` is the group name.                            |

There are magical values that can alter the behavior of the program _in the replacement string_! We control the replacement string (with the caveat that we cannot use the `/` character, since it would be stripped by the `path.basename` function). Let's see what we can use, here.

> ```javascript
> const link = "/shares/downloads/:uuid:/".replace(":uuid:", sanitizedPath);
> ```

- `$$`: We could certainly use this, though we could also write the `$` character directly, so it seems unlikely to make much of a difference.
- `$&`: In our situation, the matched substring is hardcoded to `:uuid:`. It is unclear what we could achieve with this.
- `$n`: There are no capturing groups in the matched string, this cannot be used.
- `$<name>`: There are no capturing groups in the matched string, this cannot be used.
- <code>$`</code>: Inserts the substring _before_ the match. That's <code>/shares/downloads/</code>. This does not seem particularly useful.
- `$'`: Inserts the substring _after_ the match. **Wait. That's `/`!**

**So, we can insert any number of `/` anywhere we want. `$'` is allowed in file names, so `path.basename` won't touch it.**

This leads us to our final payload.

```text
$'..$'..$'..$'..$'..$'etc$'shadow
```

`path.basename` does not modify it. Finally, the `replace` returns:
```text
/shares/downloads/../../../../../etc/shadow
```

We have achieved a path traversal!

## Conclusion

It's always difficult to find a good conclusion for these kinds of weird posts. Let's do a quick list of takeaways.

- Don't assume that the functions you're using behave as you expect them to. Especially in the JavaScript world.
- Always try to think like an attacker and block possible attempts. 
- Use proven sanitization libraries (like `path.basename`) because they handle many edge cases you probably haven't thought of.
- When you're sure that no attack is possible anymore… ask an InfoSec professional, ideally a pen-tester, to actually try it. You'll be surprised by what they find.

And, finally, if you're a library developer: _the replacement string should be inert_. If possible, avoid string-based logic entirely. For example, `String.protype.replace` can also accept a function as a replacement, which allows making arbitrary modifications. Attackers generally can't change the type of their values, so providing this (for flexibility) but not the magic values, would have made the standard library safer.

Since the JavaScript standard library cannot be versioned, this is here to stay. Forever.
