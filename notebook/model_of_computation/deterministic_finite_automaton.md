---
layout: post
title: Deterministic Finite Automaton
---

# What is Deterministic Finite Automaton (DFA)?
- DFA is a finite-state machine that accepts or rejects a given string of symbols.
- *Deterministic* refers to the uniqueness of the computation run.


## Components
- Input sequence of symbols
	- Its length is finite.
	- DFA operates on it one by one from left to right.
- State register
	- It stores the state of DFA from a finite set of states including an initial state and final states (accept states)
- Table of state transition
	- The next state is detemined by the current symbol and the current state.


## Formal definition
- DFA can be formally defined as a 5-tuple $$ M = (Q, \Sigma, \delta, q_0, F) $$ where
	- $$ Q $$ is a finite set of states
	- $$ \Sigma $$ is a finite set of input symbols
	- $$ \delta : Q \times \Sigma \to Q $$ is a transition function
	- $$ q_0 \in Q $$ is an initial state
	- $$ F \subseteq Q $$ is a set of accept states
- Let $$ w = a_1 a_2 \cdots a_n $$ be a string over $$ \Sigma $$.
- The automaton $$ M $$ accepts the string $$ w $$ if a sequence of states $$ r_0, r_1, \cdots, r_n $$ exists in $$ Q $$ with the following conditions:
	1. $$ r_0 = q_0 $$
	1. $$ r_{i + 1} = \delta (r_i, a_{i + 1})$$ for $$ i = 0, \cdots, n - 1 $$
	1. $$ r_n \in F $$



# Simple DFA Simulator in C++
Note: It might be easier to understand this implementation if you look at the examples described later at first.

```cpp
#include <iostream>
#include <string>
#include <vector>

using namespace std;

void show(const string &input, const int pos, const char state) {
	cout << endl << input << endl;
	for (int i = 0; i < pos; ++i) cout << ' ';
	cout << '^' << endl;
	cout << "state: " << state << endl;
}

int main() {
	string input;
	cout << "input the sequence of input symbols >" << endl;
	cin >> input;

	vector<string> table;
	string buffer;
	cout << endl << "input the table of state transition >" << endl;
	cin.ignore();
	while (getline(cin, buffer)) {
		if (buffer.empty()) break;
		table.push_back(buffer);
	}

	char state;
	cout << "input the initial state >" << endl;
	cin >> state;

	char accept;
	cout << endl << "input the accept state >" << endl;
	cin >> accept;

	for (int pos = 0; pos < input.size(); ++pos) {
		show(input, pos, state);

		int rule;
		for (rule = 0; rule < table.size(); ++rule) {
			if (table[rule][0] == state && table[rule][1] == input[pos]) break;
		}

		if (rule == table.size()) {
			cerr << "no corresponding state transition" << endl;
			return 0;
		}

		state = table[rule][2];
	}

	show(input, input.size(), state);
	cout << (state == accept ? "ACCEPTED" : "REJECTED") << endl;
	return 0;
}
```


## Check if the given string includes 'a'
- Input sequence of symbols: `bbbab`
- Head
	- Initial position: the head of the tape
- States
	- Initial state: `n`
	- Accept state: `y`
- Table of rules
	- Format: current state, current symbol, next state
	- `nay`
	- `nbn`
	- `yay`
	- `yby`

```
input the sequence of input symbols >
bbbab

input the table of state transition >
nay
nbn
yay
yby

input the initial state >
n

input the accept state >
y

bbbab
^
state: n

bbbab
 ^
state: n

bbbab
  ^
state: n

bbbab
   ^
state: n

bbbab
    ^
state: y

bbbab
     ^
state: y
ACCEPTED
```



# Links
- [Deterministic finite automaton - Wikipedia](https://en.wikipedia.org/wiki/Deterministic_finite_automaton)
