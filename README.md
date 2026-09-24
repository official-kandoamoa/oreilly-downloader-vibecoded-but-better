# O'Reilly epub downloader

O'Reilly provides all of their books in epub format, but only through their own
reader.

This script allows you to download all the individual files and assemble them
back into a full epub. This allows you to use other readers, e.g. for
accessibility reasons.

You need to have a valid JWT to download content. If you do not provide one,
each chapter will be cut short. You can get it by logging in with your browser
and extracting the `orm-jwt` cookie using the developer tools.

Before any usage, please read the [O'Reilly Terms of
Service](https://learning.oreilly.com/terms/).

# Usage

```
$ pip install aiohttp lxml yarl
$ touch cookies.json
$ nano cookies.json # get cookie editor from firefox extension store, install it, copy the cookies, paste it in nano and save it as cookies.json
$ python3 oreilly_downloader.py 9781633437777 --cookies cookies.json
…
created 9781633437777.epub
```

# Contributing

I am not really interested in adding any major features to this project. I will
accept fixes, but nothing that adds a significant amount of new code.

If you feel like something is missing, feel free to fork. You may also look at
rejected pull requests, maybe someone already worked on something similar.

# Similar Projects

-   <https://github.com/lorenzodifuccia/safaribooks> (python)
-   <https://github.com/hurlenko/orly> (rust)
-   <https://github.com/jenni/obooks> (javascript)
-   <https://github.com/rahulvramesh/oreilly-books-grabber> (go)
