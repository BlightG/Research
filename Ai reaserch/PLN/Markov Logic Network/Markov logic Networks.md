
[**File**](C:\Users\Gulilat Berhane\Documents\Icog\research\PLN\kbmn.pdf)
# Introduction


- **Markov Logic Networks (MLNs)** are a statistical relational learning framework that combines **first-order logic** with **probabilistic graphical models** (specifically, Markov networks). They provide a way to handle **uncertainty** in logical rules by assigning weights to formulas, making them a powerful tool for structured, uncertain domains.

- **Markov Networks**
	- Markov Networks create a probability for the values on variables that are in an undirected graph. 
	- The network applies a function over cliques (fully-connected sub graphs) 
	- the applied function will output the state of each variable that are nodes in the clique
	- we use a partition function **Z** to normalize the probability to 1.

		P(X = x) = <sup>1</sup>/<sub>Z</sub> ∏<sub>k</sub> ∅<sub>k</sub> (x<sub>{k}</sub>)
		where :
		- x<sub>{k}</sub> : is the state of the variables of the clique
		- Z = ∑<sub>x∈X</sub>  ∏<sub>k</sub>  ∅<sub>k</sub> (x<sub>{k}</sub>)

	- Alternative approach for describing the Markov network is by using a [[log-linear]] model
		P(X = x) = <sup>1</sup>/<sub>Z</sub> exp (∑<sub>j</sub> w<sub>j</sub> f<sub>j</sub>(x))
		- in changing from the first equation to the log-linear approach 
			- one feature f<sub>j</sub> matching each possible state x<sub>{k}</sub> for each clique
			- and the associated weight being w<sub>j</sub> = ∅<sub>k</sub> (x<sub>{k}</sub>)

	- Inference in Markov networks is hard because calculating exact probabilities has an exponential time complexity.
		- instead of calculating exact probabilities for inference we use approximate probabilities.
			- usually used Markov Chain Monte Carlo (MCMC)
			- Another popular method is **belief propagation** 
	- The weights of the Markov model can be found by using a gradient algorithm on the log likelihood. when using gradient algorithms we are comparing the output probability of the functions to the probability observed by the data

- **First-Order Logic**
	- A first order knowledge base is a set of sentences or formulas in [[First order logic]]
		- Formulas are constructed using four type of symbols:
			- constants: Define objects in domain of interest.
			- Variables:  These objects range over Objects in the domain.
			- Functions: Represent mappings from tuple of objects to objects.
			- Predicates: Relations among objects in the domain or attributes of objects.
		
		- A formula is *satisfiable* if and only if there exists at least one world in which it is true.
		- *Inference* in first-order-logic is only semi decidable and the rules have to be restricted to ensure it is decidable in finite time.
			- languages like prolog restrict the rules of FOL to ensure it is decidable by using such things as [[Horn clause]]

- **Markov Logic Networks**

	- a knowledge base in in FOL is used to express a set of rules that that are all true and if one fails the entire knowledge base is invalid
	- Markov logic networks seek to soften rules of first order logic by giving weights to functions based on how strong of a constraint on the world it is.
	- An MLN can be viewed as a template for creating Markov networks. 
		- Generated networks will have different sizes based on input constants but output will have the same structure.
		- [[MLN example]]
	- Formal Definition:
		- [[DEFINITION 4.1. A Markov logic network]] L is a set of pairs (Fi; wi), where Fi is a formula in first-order logic and wi is a real number. Together with a finite set of constants C = fc1; c2; : : : ; cjCjg, it defines a Markov network ML;C (Equations 1 and 2) as follows:

			1. ML;C contains one binary node for each possible grounding of each predicate appearing in L. The value of the node is 1 if the ground atom is true, and 0 otherwise.

			2. ML;C contains one feature for each possible grounding of each formula Fi in L. The value of this feature is 1 if the ground formula is true, and 0 otherwise. The weight of the feature is the wi associated with Fi in L. 

		- Assumptions for simplifying grounding of first-order-logic into Markov Logic networks
			- [[MLN ground Assumptions]]
			- Assumption 1: constants with different pattern are considered to be different
			- Assumption 2: only constants in the world are considered
			- Assumption 3: the value of a function when is a constant is known with context of that KB
		**l**