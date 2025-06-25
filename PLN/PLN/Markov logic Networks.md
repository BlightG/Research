
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
		- 