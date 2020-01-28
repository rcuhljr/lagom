---
layout: post
title: Too Many Cook's (Theorem)
categories: 
- blog
tags:
- Theory Of Computation
- Decidability
- Garey and Johnson
requires_js_math: false
---

Now that we've laid the groundwork for Cook's Theorem it's finally time to step through it. This post will follow the formalizations of the proof, while at the same time implementing the concepts in actual programs which we can run and verify our results at the conclusion of this piece. All of the code used here can be found at [this repository.](https://github.com/rcuhljr/cooks-theorem)

---

## I can't get no Satisfiability

Cook's Theorem resolves around the Satisfiability (SAT) problem, expressed as follows. Let U = {u1,u2,...,um} be a set of Boolean variables. A Truth Assignment for U is a function that maps each variable in U to either True or False. If u is a variable in U then there are two literals u and !u; u is true iff (if and only if) the truth assignment maps u to True and !u is true iff the truth assignment maps u to False. A Clause over U is a set of literals from U, for example {u1,!u3,u8}. This clause is satisfied by a truth assignment iff at least one of the members is true under the given assignment. A Collection (C) of clauses over U is satisfiable iff there exists some truth assignment for U that satisfies all clauses for C. So in a more familiar form a clause is a collection of booleans or'ed together and a collection is a set of clauses and'ed together.

We are now ready to prove that SAT is NP-Complete, and thus find the first NP-complete problem. First we'll handle the easy part of showing that SAT is in fact in NP by showing it can be verified in polynomial time. It should be apparent that an NDTM can simply guess at a truth assignment and check if that truth assignment satisfies C in polynomial time, thus the problem is verifiable in polynomial time. With that we've now passed one of the two requirements to show NP-completeness, the remaining requirement is to show that some other NP-complete problem transforms into SAT. By now the hitch should be apparent, we have no known NP-complete problems so how do we transform one into SAT? The solution is to show that all NP-complete problems transform into SAT.

Let's take a step back to our previous discussion about languages that recognize problems. There is a language that represents SAT problems with some encoding which we'll call LSAT. We want to show that all Languages in NP  transform into LSAT. While there are infinitely many Languages in NP they can all be described in one standard way, which is by giving the polynomial time NDTM that recognizes that language. So if we can find a way to represent any generic NDTM in LSAT we'll have shown that all NP Languages transform to LSAT. 

## Turing Tests

So now we have our goal laid out before us, we want to show that we can generically model an NDTM using the SAT problem thus showing that anything recognizable by an NDTM can transform into SAT. From this point out however I'm  mostly going to focus on DTM's. As we discussed previously an NDTM is almost identical to a DTM except for the guessing head which generates all possible tapes at the same time. If we can transform a DTM with it's input tape into SAT then the same non deterministic process that generates all possible input tapes could also be applied to the sat representation  to replace one specific tape with all possible permutations.

To help drive this understanding I built a series of programs representing both a DTM and the transformation from DTM into satisfiability. 

### dtm.rb
{% highlight ruby %}
  def initialize(config = {})
    @symbols = config[:symbols] || ['b', '0', '1']
    @alphabet = config[:alphabet] || ['0', '1']
    @blank = (@symbols - @alphabet).first

    @states = config[:states] || ['q0','q1','qy','qn']
    @accept = config[:accept] || 'qy'
    @reject = config[:reject] || 'qn'
    
    @transition_function = config[:transition] || {
      'q0-0'=>['q0','0',1], 'q0-1'=>['q0','1',1], 'q0-b'=>['q1','b',-1],
      'q1-0'=>['qy','b',-1], 'q1-1'=>['qn','b',-1], 'q1-b'=>['qn','b',-1],
      'qy-0'=>['qy','0',0], 'qy-1'=>['qy','1',0], 'qy-b'=>['qy','b',0],
      'qn-0'=>['qn','0',0], 'qn-1'=>['qn','1',0], 'qn-b'=>['qn','b',0]  
       }    
  end
{% endhighlight %}


Here is our constructor for our simple DTM program. I have provided a default set of values that create a simple Turing machine that recognizes even numbers in binary (does the number end in a 0?). Let's take a look at each of the parts above.  Symbols (G) is our tape alphabet, this is the characters that comprise our alphabet (T) and a blank character. States (Q) is the listing of all valid states in our Turing machine including the acceptance and rejection state. Finally we have our transition function (S) that tells you for a specific Q and G what your new Q is, what symbol to write out on the current spot before leaving, and where to move the read/write head.

Here's a short demonstration of the program running on the binary representation of 2.

    irb(main):005:0> dtm.compute(['1','0'])
          ▼
    q0:|b|1|0|b|
            ▼
    q0:|b|1|0|b|
              ▼
    q0:|b|1|0|b|
            ▼
    q1:|b|1|0|b|
          ▼
    qy:|b|1|b|b|
    => nil

So now we've seen what our DTM looks like, let's start talking about how we can represent it using SAT. The first thing we'll do is recognize that both the number of steps a DTM takes and the guessed tape length are limited to p(n), or some polynomial based on the input length (n = len(x)). It's impossible for any tape squares outside of -p(n) or p(n)+1 to be involved because the read/write head can only move one space per time increment (i) and starts at a fixed location. Similarly we only need to look at p(n)+1 distinct times that need to be considered. For our example above we can see that just saying p(n) = 2*n+1 is more than sufficient, as all the algorithm does is transverse the string once and then check the last character. So now that we have a bound p(n) it will also be useful to have some other bounds for our state machine. Well will use r as the bound for the number of states (r = len(Q)-1) and v as the bound for our tape alphabet (v = len(G)-1). Using these bounds let's now look at the variables we'll want to use in our SAT transformation.

  
    Q[i,k]   0<=i<=p(n)  0<=k<=r

This variable simply says that at time i, the machine is in state Qk, which is why k is bounded by r (number of states) and i will be bounded by p(n) in all of these variables.

    H[i,j]   0<=i<=p(n)  -p(n)<=j<=p(n)+1

This variable enforces that at time i the read-write head is scanning tape square j. In some of the examples I may make the lower bound of J to be zero instead of -p(n) for simplicity as our example DTM never moves into the negatives and it doesn't affect the order of the run time.

    S[i,j,k]  0<=i<=p(n)  -p(n)<=j<=p(n)+1  0<=k<=v

This variable says that at time i the contents of tape square j is the symbol Sk, note that k is bounded by the size of our alphabet (v) here.

Now we need to build up a set of clauses that is satisfiable iff the truth assignment represents a machine accepting the input string in under p(n) steps. Once we've done that we'll just need to show that the generation of the clauses from the DTM can be accomplished in polynomial time and we'll have shown that the transformation is valid.


## Clause Groupies

There are six clause groups that we will assemble to accomplish this proof, and so we'll make a program that takes our DTM from earlier and generates those six groups from it. As a note, for ease of parsing my SAT clauses are written using spaces to separate variables and commas to separate clauses. For this reason I replace commas in variables with semicolons. When reading variables in my example code Q[0,0] is equivalent to Q[0;0].

{% highlight ruby %}
  def initialize(dtm = nil)
    @dtm = dtm || DTM.new()     
    @pn = 5 #input length factor
    @r = dtm.states.length-1
    @v = dtm.symbols.length-1
  end
{% endhighlight %}

You can see where our definitions match what we described earlier. We're going to test this with a very short string so 5 is a sufficient number of steps for the machine to resolve.

## Clause Group 1

The goal of this group is to imply the restriction that the Machine (M) is in exactly one state at every time (i). This requires two kinds of clauses, the first p(n)+1 clauses will specify that the machine is in at least one state.

    {Q[i,0],Q[i,1],...,Q[i,r]} 0<=i<=p(n)

This will result in clause groups generated that look like this

    Q[0;0] Q[0;1] Q[0;2] Q[0;3]
    Q[1;0] Q[1;1] Q[1;2] Q[1;3]

where it's easy to see how the machine must be in at least one state for each time i.

The remaining (p(n)+1)(r+1)(r/2) clauses must be designed to so that they are only satisfiable if at no time i is M ever in more than one state.

    {!Q[i,j], !Q[i,j']}   0<=i<=p(n) 0<=j<j'<=r

Let's look at what some of these clauses look like when we transform our DTM.

    !Q[0;0] !Q[0;1]
    !Q[0;0] !Q[0;2]
    !Q[0;0] !Q[0;3]
    !Q[0;1] !Q[0;2]
    !Q[0;1] !Q[0;3]
    !Q[0;2] !Q[0;3]

Note that if Q[0,0] is true then that forces every other time zero state to be false to satisfy the first three clauses, similarly if Q[0,2] is true then state three is forced false as are states 0 and 1 from the second and fourth line.

Now let's look at the code that actually generates this group.

{% highlight ruby %}
  def clause_group_one    
    result = (0..@pn).map{ |i| 
                (0..@r).map{ |r| 
                  "Q[#{i};#{r}]" 
                }.join(" ")  
              }.join(", ")
    result + ", " + (0..@pn).map{ |i| 
                      (0..@r-1).map{ |j| 
                        (j+1..@r).map{ |j_prime| 
                          "!Q[#{i};#{j}] !Q[#{i};#{j_prime}]" 
                        }.join(", ") 
                      }.join(", ") 
                    }.join(", ")
  end
{% endhighlight %}

These should be fairly straightforward mappings from the descriptions I gave above, for those not familiar with Ruby the syntax #{i} when used inside a string evaluates i and inserts the result into the string at that location.

## Clause Group 2 and 3

These two groups are similar in construction to clause group 1. Group two asserts that the read write head is scanning exactly one square at each time i and group three asserts that each square of the tape contains exactly one symbol from G at each time i. In both of these examples I limit tape locations to positive indices for simplicity.

    {H[i,0],H[i,1],...,H[i,p(n)+1]}  0<=i<=p(n)
    {!H[i,j], !H[i,j']}              0<=i<=p(n) 0<=j<j'<=p(n)+1

    {S[i,j,0],S[i,j,1],...,S[i,j,v]}  0<=i<=p(n) 0<=j<=p(n)+1 
    {!S[i,j,k], !S[i,j,k']}           0<=i<=p(n) 0<=j<=p(n)+1 0<=k<k'<=v

You'll remember from earlier that v is the number of symbols in our alphabet. The code and generated outputs are quite similar to those above so I'll leave them out for brevities sake, all of the materials are available in the repository mentioned in the introduction.

## Clause Group 4 and 5

These two groups are very straight forward and consist entirely of single literal clauses. Group 4 specifies the starting conditions of the machine, namely by creating clauses like

    Q[0;0], H[0;0], S[0;0;1], S[0;1;0]

we've forced the machine to start at state zero, with the read head at slot zero, and the contents of the tape being '0b' (zero followed by a blank character).

Group five defines our end state, we create the clause

    Q[p(n),2]
  
Which specifies that by the end of the run time it must be in our acceptance state (if you examine our state list in the code above, qy is in index 2).

## Clause Group 6

Clause group 6 is a bit more complex as this is the group that enforces that every state of computation follows from the previous one by a single step of the program M. Let's look at two distinct sub groups of group 6 the first of which guarantees that if the read write head is not scanning a tape square j at time i, then the symbol in square j does not change between times i and i+1. Let's look at the formal expression and an example clause.

    {!S[i,j,l] H[i,j] S[i+1,j,l]} 0<=i<=p(n) 0<=j<=p(n)+1 0<=l<=v

    !S[0;0;0] H[0;0] S[1;0;0]

The key to understanding these expressions is to learn to look for what causes the statement to be unsatisfiable. This is going to get a little heavy on the double negatives so step through it until it really becomes clear. In order for the first literal ![S;0;0;0] to be false S[0;0;0] must be true, so at time zero slot zero there must have been a zero written in. In order for the third literal S[1;0;0] to be false variable S[1;0;0] must be false. From our earlier clauses however we know some S[1;0;N] must be true (some symbol exists in that slot) so some symbol other than zero has to be in slot 0 at time 1. We already said that at time one slot zero had zero written in so we can see it must have changed. From that point we can see the only way for the statement to fail is for H[0;0] to be false, or that the head was in any slot besides zero at time zero. Thus the only way theses clauses are satisfied will be when the character in a slot on the tape stays the same between time steps, or the read head had to be in that square at the time of the change.

As we're keeping an eye on this transformation to make sure it doesn't exceed polynomial time it can be seen that the previous sub group adds 2(v+1)(p(n)+1)^2 clauses, so still polynomial time bounded by O(p(n)^2).

The final subgroup of 6 guarantees that changes from one configuration to the next are done according to our transition function for M. We construct a triple of clauses for each item in the set (i,j,k,l).

    0<=i<=p(n) 0<=j<=p(n)+1 0<=k<=r 0<=l<=v
    {!H[i,j] !Q[i,k] !S[i,j,l] H[i+1,j+delta]}
    {!H[i,j] !Q[i,k] !S[i,j,l] Q[i+1,k']}
    {!H[i,j] !Q[i,k] !S[i,j,l] S[i+1,j,l']}

Delta, k' and l' are the results of our transition function for the given Qk (state) and Gl(symbol). In our code this looks like this.

{% highlight ruby %}
    (0..@pn-1).map{ |i| 
      (0..@pn+1).map{ |j| 
        (0..@r).map{ |k| 
          (0..@v).map{ |l|

      k_prime, l_prime, delta = translate_dtm_step(k,l)

      "!S[#{i};#{j};#{l}] !H[#{i};#{j}] !Q[#{i};#{k}] H[#{i+1};#{j+delta}]" + ", " +
      "!S[#{i};#{j};#{l}] !H[#{i};#{j}] !Q[#{i};#{k}] Q[#{i+1};#{k_prime}]" + ", " +
      "!S[#{i};#{j};#{l}] !H[#{i};#{j}] !Q[#{i};#{k}] S[#{i+1};#{j};#{l_prime}]"
      
    }.join(", ") }.join(", ") }.join(", ") }.join(", ")


  def translate_dtm_step(k, l)
    k_prime, l_prime, delta = @dtm.next_step(@dtm.states[k], @dtm.symbols[l])
    k_prime = @dtm.states.index k_prime
    l_prime = @dtm.symbols.index l_prime
    [k_prime, l_prime, delta]
  end
{% endhighlight %}


    !S[0;0;1] !H[0;0] !Q[0;0] H[1;1]
    !S[0;0;1] !H[0;0] !Q[0;0] Q[1;0]
    !S[0;0;1] !H[0;0] !Q[0;0] S[1;0;1]


Look at the small subset of generated results, the way in which this can be rendered unsatisfiable is if the contents of the tape at time 0 was index 1, the head was reading slot zero, and the state was zero all of which having to be true (thus we've fully defined the state of the machine in time zero) while at the next time increment (1) either the Head didn't move one right, or the state didn't stay zero, or the head didn't leave the contents of the slot at index 1. So if at any time a transition occurs that doesn't match the results specified by the transition function the clause group is rendered unsatisfiable.

This clause group contains 6(p(n))(p(n)+1)(r+1)(v+1) clauses thus finishing our verification that we can generate the SAT from the DTM in polynomial time.

## Putting it all together

We can now bundle all of these clause groups together. For our very simplified problem state of a string containing one character '0' and a DTM that determines if the string ends in 0 we end up with about 220 unique variables and 1700 clause groups. If we used the naive brute force solver I implemented in sat.rb with a SAT problem this complex it would never finish. Thankfully there are much more robust solvers available. I ended up using [MiniSat](http://minisat.se/) as my solver so after handing our SAT translation of the state machine over to MiniSat it solves it in under a second. This is certainly an interesting throwback to our initial discussion about NP hard problems and finding faster solutions. MiniSat gives us one satisfying arrangement of our machine which we can then look at to see if it makes sense.

    Q[Time;State]
    Q[0;0]
    Q[1;0]
    Q[2;1]
    Q[3;2]

    H[Time;Location]
    H[0;0]
    H[1;1]
    H[2;0]
    H[3;-1]

    S[Time;Location;Symbol]
    S[0;0;1]
    S[0;1;0]
    S[1;0;1]
    S[1;1;0]
    S[2;0;1]
    S[2;1;0]
    S[3;0;0]
    S[3;1;0]

So we can see that the machine starts in q0, moves to q1 at time 1 and moves into the accept state by time 3. The read head starts at zero, moves right, finds the blank, and moves left two more times just like we'd expect. If we look at the symbols in the tape we can see that it contains '0b' up until the zero gets replaced with a blank as part of state q1's behavior at time 3. If we so desired we can invert the found solution and add it in as another clause forcing the solver to find another unique solution at which point it will fail as there is only one valid navigation for the machine we've specified.

Now that we've seen that we can transform a generic DTM into a SAT problem, and we've shown that the generated SAT problems size is bounded by O(p(n)^2) we have a polynomial time transformation for all NP problems into SAT thus finishing the proof that SAT is NP-Complete.

