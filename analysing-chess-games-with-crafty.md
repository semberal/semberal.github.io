# Analysing chess games with Crafty
> 26/02/2012

Since my final migration from Windows to Linux, I've been searching for a decent alternative to Fritz, which used to be my favorite chess analysis software. Because Chessbase is [obviously not interested](http://www.chessbase.com/newsdetail.asp?newsid=3189) in creating Linux versions of their products, I had to search for an alternative. Thankfully there is a lot of open-source chess engines for Linux, with [Crafty](http://www.craftychess.com/) being the best known one. Crafty is developed as a research project led by [Dr. Robert Hyatt](http://www.cis.uab.edu/info/faculty/hyatt/hyatt.html). It's [ELO](http://en.wikipedia.org/wiki/Elo_rating_system) is not among the highest (~2600), but since I'm just a hobby player, it perfectly suits my needs. If you are looking for the most powerful chess engines available, though, take a look at [Stockfish](http://www.stockfishchess.com/),  [Houdini](http://www.cruxis.com/chess/houdini.htm) or [Rybka](http://www.rybkachess.com).

I've been using chess engines for analyses of my games which I play either at my favorite [Free Internet Chess Server](http://www.freechess.org/) or occasionally at [Chess.com](http://www.chess.com/). After finishing the game (both winning or losing one), I use a chess engine to annotate it; show me blunders and offer alternative moves. In this blog post, I'll show how to do it with Crafty.

To unleash the full power of Crafty, it's necessary to configure it to use opening books and endgame tablebases properly. Dr. Hyatt provides all the required files at his [FTP server](ftp://ftp.cis.uab.edu/pub/hyatt/).

To create an opening book, we need a database of high-quality games in PGN format. The larger the database is, the better results we'll get. So, download `start.pgn` and all the `enormous*.zip` files from the [FTP server](ftp://ftp.cis.uab.edu/pub/hyatt/pgn). Once the zip file is downloaded and extracted, start crafty and type:

```
White(1): book create enormous.pgn 60
White(1): books create start.pgn 60
```

More detailed explanation of the commands and their parameters can be found in the [Crafty documentation](http://www.cis.uab.edu/hyatt/craftydoc.html#Opening%20Book).

Now move both `book.bin` and `books.bin` to the directory where you will store opening books. It can be directory of your choice, I decided for `~/.crafty/books`.

To obtain endgame tablebases, open Dr. Hyatt's FTP server again and download all the files from [`TB/3-4-5`](ftp://ftp.cis.uab.edu/pub/hyatt/TB/3-4-5) and [`TB/tbs`](ftp://ftp.cis.uab.edu/pub/hyatt/TB/tbs) subdirectories (it's quite a lot of files there and around 7GB of data, you can use `wget` or [DownThemAll!](https://addons.mozilla.org/en-US/firefox/addon/downthemall/) Firefox extension to batch-download it) and store them again to some directory of your choice (`~/.crafty/TB` in my case).

The final step is to create the Crafty configuration file `~/.craftyrc`. There is a lot configuration options, which are in detail explained in the [documentation](http://www.cis.uab.edu/hyatt/craftydoc.html). Here is the configuration I'm using:

```
#
# Crafty chess engine configuration
#
smpmt=4 ## number of cores used by Crafty
hash=1024M ## size of the transposition/refutation hash table
hashp=256M ## size of the pawn structure/king safety hash table
#ponder=on ## think at opponent's time
swindle=on ## try to win drawn games (according to egtb)
learn=7 ## book, position and result learning
bookpath=/home/semberal/.crafty/books ## opening books path
egtb=on ## use endgame tablebases
tbpath=/home/semberal/.crafty/TB ## tablebases path
logpath=/home/semberal/.crafty/logs ## where to put log files (log.001, game.001)
exit
```

Keep in mind that Crafty for some reasons doesn't understand paths with `~`, so all paths have to be absolute! Test your configuration by launching Crafty. If you see no errors and the displayed configuration values match those you've entered into `.craftyrc`, you've set everything up correctly.

Let's try to do some analysis now. Start Crafty in a directory with your PGN games and type:

```
White(1): annotate mygame.pgn bw 1-999 0.3 30
```

This means that we would like to annotate all moves (1-999) of `mygame.pgn`, for both sides (bw), with margin 0.3 and 30 seconds of computer time per half-move (see the [Crafty documentation](http://www.cis.uab.edu/hyatt/craftydoc.html) for a detailed explanation of all the parametrers). Crafty will (in this case) produce a file `mygame.pgn.can` which can be loaded into some graphical interface (such as my favorite [Scidb](http://scidb.sourceforge.net/)) and reviewed.

I hope the article was helpful with first-time Crafty configuration. If you run into problems, don't hesitate to ask in the discussion.
