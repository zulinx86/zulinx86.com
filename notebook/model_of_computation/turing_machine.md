---
layout: post
title: Turing Machine
---

## What is Turing Machine?
A Turing machine mathematically models a machine that mechanically operates on a tape.


### Components
- Tape
	- Its length is infinite.
	- It is divided into discrete cells, one next to the other.
	- Each cell contains a symbol from a finite set of symbols including a special blank symbol.
- Head
	- It can move cells on the tape.
	- It can read and write the symbol of the cell where the head is.
- State register
	- It stores the state of the Turing machine from a finite set of states including an initial state and final states.
	- When the state goes to the final states, the machine halts.
- Table of rules / instructions
	- Given the current state and the symbol currently under the head, the table returns the following items:
		1. New symbol currently under the head
		1. New position of the head (one step left, one stpe right, or stay in the same place)
		1. New state of the machine
	- It is called 5-tuple models because it has two inputs and three outputs.


### Formal Definition
- Turing machine can be formally defined as a 7-tuple $$ M = \langle Q, \Gamma, b, \Sigma, \delta, q_0, F \rangle $$ where
	- $$ \Gamma $$ is a finite, non-empty set of tape alphabet symbols
	- $$ b \in \Gamma $$ is the blank symbol (the only symbol allowed to occr on the tape infinitely)
	- $$ \Sigma \subseteq \Gamma \setminus \{ b \} $$ is the set of input symbols, that is, the set of symbols allowed to appear in the initial tape contents
	- $$ Q $$ is a finite, non-empty set of states
	- $$ q_0 \in Q $$ is the initial state
	- $$ F \subseteq Q $$ is the set of final states or accepting states
	- $$ \delta : (Q \setminus F) \times \Gamma \mapsto Q \times \Gamma \times \{ L, R, S \} $$ is a transition function, where $$ L $$ is left shift, $$ R $$ is right shift, $$ S $$ is stay.



## Simple Turing Machine Simulator in C++
Note: It might be easier to understand this implementation if you look at the examples described later at first.

```cpp
#include <iostream>
#include <vector>
#include <string>

using namespace std;

void show(const string &tape, const int head, const char state) {
	cout << endl << "...-" << tape << "-..." << endl;
	for (int i = 0; i < head + 4; ++i) cout << ' ';
	cout << '^' << endl;
	for (int i = 0; i < head + 4; ++i) cout << ' ';
	cout << state << endl;
}

int main() {
	string tape;
	cout << "input the initial tape >" << endl;
	cin >> tape;

	vector<string> table;
	string buf;
	cout << endl << "input the table of rules >" << endl;
	cin.ignore();
	while (getline(cin, buf)) {
		if (buf.empty()) break;
		table.push_back(buf);
	}

	int head;
	cout << "input the inital head position >" << endl;
	cin >> head;

	char state;
	cout << endl << "input the initial state >" << endl;
	cin >> state;

	while (true) {
		show(tape, head, state);

		if (state == 'H') break;

		int rule;
		for (rule = 0; rule < table.size(); ++rule) {
			if (table[rule][0] == state && table[rule][2] == tape[head])
				break;
		}

		if (rule == table.size()) {
			cerr << "no corresponding rules in the given table" << endl;
			return 1;
		}

		state = table[rule][1];
		tape[head] = table[rule][3];
		switch (table[rule][4]) {
		case 'L':
			--head;
			break;
		case 'R':
			++head;
			break;
		case 'S':
			break;
		}

		if (head == -1) {
			tape = '-' + tape;
			head = 0;
		} else if (head == tape.size()) {
			tape += '-';
		}
	}

	return 0;
}
```


### Change `0` to `1` until the head reaches `-`
- Tape
	- Initial tape: `01100`
	- Final tape: `11111`
- Head
	- Initial position: the head of the tape
- States
	- Initial state: `A`
	- Final state: `H`
- Table of rules
	- Format: current state, next state, current symbol, next symbol, head position
	- `AA01R`
	- `AA11R`
	- `AH--S`

```
input the initial tape >
01100

input the table of rules >
AA01R
AA11R
AH--S

input the inital head position >
0

input the initial state >
A

...-01100-...
    ^
    A

...-11100-...
     ^
     A

...-11100-...
      ^
      A

...-11100-...
       ^
       A

...-11110-...
        ^
        A

...-11111--...
         ^
         A

...-11111--...
         ^
         H
```


### Addition of two binary numbers
- Objective: add `0b1010` and `0b111`
- Tape
	- Initial tape: `1010-111`
	- Final tape: `10001-000`
- Table of rules
	- Go until `-`
		- `AA00R`
		- `AA11R`
	- Go beyond `-`
		- `AB--R`
	- If all the right-side values are `0`, the system halts.
		- `BB00R`
		- `BH--S`
		- `BC11R`
		- `CC00R`
		- `CC11R`
		- `CD--L`
	- Decrement the right value
		- `DE10L`
		- `DD01L`
	- Go beyond `-`
		- `EE00L`
		- `EE11L`
		- `EF--L`
	- Increment the left value
		- `FA01R`
		- `FF10L`
		- `FA-1S`

```
input the initial tape >
1010-111

input the table of rules >
AA00R
AA11R
AB--R
BB00R
BH--S
BC11R
CC00R
CC11R
CD--L
DE10L
DD01L
EE00L
EE11L
EF--L
FA01R
FF10L
FA-1S

input the inital head position >
0

input the initial state >
A

...-1010-111-...
    ^
    A

...-1010-111-...
     ^
     A

...-1010-111-...
      ^
      A

...-1010-111-...
       ^
       A

...-1010-111-...
        ^
        A

...-1010-111-...
         ^
         B

...-1010-111-...
          ^
          C

...-1010-111-...
           ^
           C

...-1010-111--...
            ^
            C

...-1010-111--...
           ^
           D

...-1010-110--...
          ^
          E

...-1010-110--...
         ^
         E

...-1010-110--...
        ^
        E

...-1010-110--...
       ^
       F

...-1011-110--...
        ^
        A

...-1011-110--...
         ^
         B

...-1011-110--...
          ^
          C

...-1011-110--...
           ^
           C

...-1011-110--...
            ^
            C

...-1011-110--...
           ^
           D

...-1011-111--...
          ^
          D

...-1011-101--...
         ^
         E

...-1011-101--...
        ^
        E

...-1011-101--...
       ^
       F

...-1010-101--...
      ^
      F

...-1000-101--...
     ^
     F

...-1100-101--...
      ^
      A

...-1100-101--...
       ^
       A

...-1100-101--...
        ^
        A

...-1100-101--...
         ^
         B

...-1100-101--...
          ^
          C

...-1100-101--...
           ^
           C

...-1100-101--...
            ^
            C

...-1100-101--...
           ^
           D

...-1100-100--...
          ^
          E

...-1100-100--...
         ^
         E

...-1100-100--...
        ^
        E

...-1100-100--...
       ^
       F

...-1101-100--...
        ^
        A

...-1101-100--...
         ^
         B

...-1101-100--...
          ^
          C

...-1101-100--...
           ^
           C

...-1101-100--...
            ^
            C

...-1101-100--...
           ^
           D

...-1101-101--...
          ^
          D

...-1101-111--...
         ^
         D

...-1101-011--...
        ^
        E

...-1101-011--...
       ^
       F

...-1100-011--...
      ^
      F

...-1110-011--...
       ^
       A

...-1110-011--...
        ^
        A

...-1110-011--...
         ^
         B

...-1110-011--...
          ^
          B

...-1110-011--...
           ^
           C

...-1110-011--...
            ^
            C

...-1110-011--...
           ^
           D

...-1110-010--...
          ^
          E

...-1110-010--...
         ^
         E

...-1110-010--...
        ^
        E

...-1110-010--...
       ^
       F

...-1111-010--...
        ^
        A

...-1111-010--...
         ^
         B

...-1111-010--...
          ^
          B

...-1111-010--...
           ^
           C

...-1111-010--...
            ^
            C

...-1111-010--...
           ^
           D

...-1111-011--...
          ^
          D

...-1111-001--...
         ^
         E

...-1111-001--...
        ^
        E

...-1111-001--...
       ^
       F

...-1110-001--...
      ^
      F

...-1100-001--...
     ^
     F

...-1000-001--...
    ^
    F

...--0000-001--...
    ^
    F

...-10000-001--...
    ^
    A

...-10000-001--...
     ^
     A

...-10000-001--...
      ^
      A

...-10000-001--...
       ^
       A

...-10000-001--...
        ^
        A

...-10000-001--...
         ^
         A

...-10000-001--...
          ^
          B

...-10000-001--...
           ^
           B

...-10000-001--...
            ^
            B

...-10000-001--...
             ^
             C

...-10000-001--...
            ^
            D

...-10000-000--...
           ^
           E

...-10000-000--...
          ^
          E

...-10000-000--...
         ^
         E

...-10000-000--...
        ^
        F

...-10001-000--...
         ^
         A

...-10001-000--...
          ^
          B

...-10001-000--...
           ^
           B

...-10001-000--...
            ^
            B

...-10001-000--...
             ^
             B

...-10001-000--...
             ^
             H
```



## What is Universal Turing Machine and Turing-complete?
- A universal turing machine (UTM) is a Turing machine that simulates an arbitrary Turing machine on arbitrary input.
- A system that can simulate a universal Turing machine is called Turing complete.



## Links
- [Turing machine - Wikipedia](https://en.wikipedia.org/wiki/Turing_machine)
- [Universal Turing machine - Wikipedia](https://en.wikipedia.org/wiki/Universal_Turing_machine)
- [Turing completeness - Wikipedia](https://en.wikipedia.org/wiki/Turing_completeness)
- [万能チューリングマシンの創り方 - Qiita](https://qiita.com/38912Pataro/items/ffdc549b3b8fa3c43325)