---
layout: post
title:  "How Playing Tic Tac Toe Saved My Startup"
date:   2013-12-19
---

I'm going to teach you how to play tic tac toe! Namely, what the best first move to make is. You may have heard that certain starting moves are better than others, but have you ever been shown proof of which move is optimal? We'll build up an AI from first principles, and make calculate the best first move. Why, you ask? Because why not.

##How Will It Work?

It'll be pretty simple. We'll write code that given a board and whose turn it is, calculate how to:

1. Win: Complete a line to get three in a row immediately.
2. Block: Stop the opponent from getting three in a row immediately.
3. Plan: Start a chain of moves that guarantee a win.
4. Defend: Block a chain of moves that guarantee a loss.
5. Calculate: Make a move that is most likely to trigger a guaranteed win.

##Board Representation

Our tic tac toe board will be represented by two sets of 9 bits.

{% highlight c++ %}
// Each box has the following bit position
987
654
321

// To represent player 'x' filling up the top row, we have
// (0b represents a binary literal)
short topRow = BINARY(111000000);

// Lets define a few patterns we'll use later
const short UPPER_LEFT = BINARY(100000000);

const short FULL_BOARD = BINARY(111111111);

const short HORIZ_0 = BINARY(111000000);
const short HORIZ_1 = BINARY(000111000);
const short HORIZ_2 = BINARY(000000111);

const short VERT_0 = BINARY(100100100);
const short VERT_1 = BINARY(010010010);
const short VERT_2 = BINARY(001001001);

const short DIAG_0 = BINARY(100010001);
const short DIAG_1 = BINARY(001010100);
{% endhighlight %}

The nice thing about this representation is that it's easy to operate on using bit masking. Lets define a few utility functions.

{% highlight c++ %}
// Check if we have a winner!
bool didPlayerWin(short moves) {
  if (((moves & HORIZ_0) == HORIZ_0) ||
      ((moves & HORIZ_1) == HORIZ_1) ||
      ((moves & HORIZ_2) == HORIZ_2) ||
      ((moves & VERT_0)  == VERT_0)  ||
      ((moves & VERT_1)  == VERT_1)  ||
      ((moves & VERT_2)  == VERT_2)  ||
      ((moves & DIAG_0)  == DIAG_0)  ||
      ((moves & DIAG_1)  == DIAG_1))
    return true;
  return false;
}

// Where are we allowed to move?
short validMoves(short xMoves, short oMoves) {
  return ~(xMoves | oMoves) & FULL_BOARD;
}

// Board visualizer
void printBoard(string description, short xMoves, short oMoves) {
  cout << description << endl;
  for (int row=0; row<3; row++) {
    for (int col=0; col<3; col++) {
      short currentPosition = UPPER_LEFT >> (col + row*3);
      if ((xMoves & currentPosition) != 0)
        cout << "x";
      else if ((oMoves & currentPosition) != 0)
        cout << "o";
      else
        cout << "-";
    }
    cout << endl;
  }
}

// Is it over yet?
bool gameOver(short xMoves, short oMoves) {
  if ((validMoves(xMoves, oMoves) & FULL_BOARD) == 0)
    return true;
  return didPlayerWin(xMoves) || didPlayerWin(oMoves);
}
{% endhighlight %}

##A Quick Win/Loss

Lets implement the first step of our AI, winning in the current move. This is pretty straightforward, we just check over every move, and get all the ones that complete a line.

{% highlight c++ %}
// if we move here, we'll win immediately!
short winningMoves(short thinkingPlayer, short idlePlayer) {
  if (gameOver(thinkingPlayer, idlePlayer))
    return 0;
  short moves = validMoves(thinkingPlayer, idlePlayer);
  short winning = 0;
  // for all valid moves..
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel) {
      // check if moving here makes us win
      if (didPlayerWin(thinkingPlayer | sel))
        winning |= sel;
    }
  }
  return winning;
}
{% endhighlight %}

The second step of our AI is blocking our opponent from winning. We'll implement this in a similar fashion, just checking over every move to see if it gives our opponent a winning move on the next turn.

{% highlight c++ %}
// if we move here, we'll lose next turn!
short losingMoves(short thinkingPlayer, short idlePlayer) {
  if (gameOver(thinkingPlayer, idlePlayer))
    return 0;
  short moves = validMoves(thinkingPlayer, idlePlayer);
  short trapping = 0;
  // for all valid moves..
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel) {
      // check if moving here gives the opponent a winning move
      if (winningMoves(idlePlayer, thinkingPlayer | sel))
        trapping |= sel;
    }
  }
  return trapping;
}
{% endhighlight %}

##Planning A Step Ahead

We want get into a position where we will always win, reguardless of what moves our opponent makes. If you've played tic tac toe before, this will be familiar to you. It's where you have two opportunities of completing a line.

{% highlight c++ %}
// o's turn
// no matter where o goes, x will win
// they can't block both of the almost complete lines
x-x  
-xo  
o--
{% endhighlight %}

So how do we detect these situtations? We can determine if a move will put us in one of these situations by pretending to make the move and iterating through each of our opponent's possible next moves, checking if we have a winning move for each of them.

{% highlight c++ %}
// if we move here, we'll win right now or on our next turn
short planWinningMoves(short thinkingPlayer, short idlePlayer) {
  if (gameOver(thinkingPlayer, idlePlayer))
    return 0;
  short moves = validMoves(thinkingPlayer, idlePlayer);
  short winning = 0 | winningMoves(thinkingPlayer, idlePlayer);
  // for all valid moves..
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel) {
      short opponentMoves =
          validMoves(idlePlayer, thinkingPlayer | sel);
      if (losingMoves(idlePlayer, thinkingPlayer | sel) ==
          opponentMoves) {
        // moving here forces the opponent to make a bad move
        winning |= sel;
      }
    }
  }
  return winning;
}
{% endhighlight %}

Now that we know how to detect moves that lead to guaranteed wins, we can also find out which moves let our opponent make those moves.

{% highlight c++ %}
// if we move here, our opponent can plan a winning move
short planLosingMoves(short thinkingPlayer, short idlePlayer) {
  if (gameOver(thinkingPlayer, idlePlayer))
    return 0;
  short moves = validMoves(thinkingPlayer, idlePlayer);
  short trapping = 0 | losingMoves(thinkingPlayer, idlePlayer);
  // for all valid moves..
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel) {
      // check if moving here guarantees a win for them
      if (planWinningMoves(idlePlayer, thinkingPlayer | sel))
        trapping |= sel;
    }
  }
  return trapping;
}
{% endhighlight %}

##Two Steps Ahead

Now let's do the same thing, except planning two steps ahead! Turns out for tic tac toe, the furthest you ever need to look ahead is by two steps.

{% highlight c++ %}
// now we want perfect moves
// aka moves that start a chain where we win for sure
short perfWinningMoves(short thinkingPlayer, short idlePlayer) {
  if (gameOver(thinkingPlayer, idlePlayer))
    return 0;
  short moves = validMoves(thinkingPlayer, idlePlayer);
  short winning = 0 | winningMoves(thinkingPlayer, idlePlayer);
  // for all valid moves..
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel) {
      // check if moving here forces them to make a bad move
      if (planLosingMoves(idlePlayer, thinkingPlayer | sel) ==
			   validMoves(idlePlayer, thinkingPlayer | sel)) {
        winning |= sel;
      }
    }
  }
  return winning;
}
{% endhighlight %}

And similarly for predicting moves that get let our opponent make a perfect move.

{% highlight c++ %}
// we also want perfectly bad moves
// aka moves that start a chain where we lose for sure
short perfLosingMoves(short thinkingPlayer, short idlePlayer) {
  if (gameOver(thinkingPlayer, idlePlayer))
    return 0;
  short moves = validMoves(thinkingPlayer, idlePlayer);
  short trapping = 0 | losingMoves(thinkingPlayer, idlePlayer);
  // for all valid moves..
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel) {
      // check if moving here lets them make a winning move
      if (perfWinningMoves(idlePlayer, thinkingPlayer | sel))
        trapping |= sel;
    }
  }
  return trapping;
}
{% endhighlight %}

Looks good! Now it's time to put all these functions to use. Let's see which moves lead to a guaranteed loss in a few situations.

{% highlight c++ %}
//O's turn
---
-x-
---
//If O moves here, he will lose
-x-
x-x
-x-

//O's turn
x--
---
---
//If O moves here, he will lose
-xx
x-x
xxx
{% endhighlight %}

So if x decides to move in the middle their first turn, 4/8 of the available moves will load to a definite loss, but if they decide to move in the corner, 7/8 of the available moves will lead to a definite loss!

##Calculating All the Moves

Lets make similar calculations for all the first moves we can make! We'll see how many possible moves there are for our opponent, and how many of those lead us to a guaranteed win.

{% highlight c++ %}
int countMoves(short moves) {
  int result = 0;
  for (short sel = 1 << 8; sel != 0; sel = sel >> 1) {
    if (moves & sel)
      result++;
  }
  return result;
}

void printFirstMoveTable() {
  for (int row=0; row<3; row++) {
    for (int col=0; col<3; col++) {
      short move = UPPER_LEFT >> (col + row*3);

      short availableMoves = validMoves(0, move);
      short badMoves = perfLosingMoves(0, move);

      cout << setiosflags(ios::fixed) << setprecision(3);
      cout << countMoves(badMoves)/
      	(double)countMoves(availableMoves);
      if (col != 2)
        cout << " | ";
    }
    cout << endl;
    if (row != 2)
      cout << "---------------------" << endl;
  }
}
{% endhighlight %}

Running the code gives us the following table.

{% highlight c++ %}
0.875 | 0.500 | 0.875
---------------------
0.500 | 0.500 | 0.500
---------------------
0.875 | 0.500 | 0.875
{% endhighlight %}

##Conclusion

So now you know, if you're ever playing tic tac toe with someone, go for the corners. If they don't know what they're doing, odds are they won't move in the center and you'll win for sure! You can grab the full code listing [here](https://gist.github.com/a12x/8038716).
