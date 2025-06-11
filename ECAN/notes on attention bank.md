## description and design issues
- the module holds implementation of "attention bank" -- a way of allocating and distributing fixed amounts of attention values to various different atoms. the attention value implies the importance of an atom, for various algorithms.
## goal
- it is to minimize the impacts of combinatorial explosion of possibilities in various algorithms.
- the main idea is that the most important atoms should be examined first, and that importance can be spread algorithmically to control the exploration of various branches.
- The general idea is that the attention-span is limited, and that most computation happens near where the attention is distributed. Because the supply of money is limited, it is treated like "money", that is, economically. The bank is the subystem that keeps track of how much of it there is. (from bank's readme)
## design issues
#### different flavors
- there is only one kind of attention value described in the attention bank module.
#### callbacks
- the design at some point seemed a bit awkward as attention values were values hanging off of atoms and users could directly access them. However, when they wanted to set values they had to do it through the attention bank as otherwise the bank won't know of the changes. What has been done to resolve this issue? using callbacks to notify the attention bank for changes in attention values.
- Attention bank subscribes to callbacks or notifiers. The callbacks need to be registered in a centralized place. where should this centralized place be? 
	- the attention bank it self
	- localized to atoms in the atomspace
	- globally (for all atoms)
	the signals seem to be defined in the AttentionBank class as AFCHSigl and AVCHSigl signals.
- using callbacks to notify the attention bank to register any changes on attention values is stated as a possibility
#### attention focus pattern matcher
- it holds an implementation for a pattern matcher extension that looks at the attention focus boundary during the pattern matching search. A Question of whether this is needed is raised.
- the same question can be found inside AFImplicator.cc line 25

### Description of class members under attention focus class
#### class name AttentionBank, file AttentionBank.h
- this is like a set but can hold duplicated values of pair<handle, attetionptr> and takes a comparing function to sort its elements i.e compare_sti_less
```cpp
std::multiset<std::pair<Handle, AttentionValuePtr>, compare_sti_less> attentionalFocus;
```
- it uses mutexs to ensure the sti and lti values are changed synchronously i.e changes being performed do not interfere with each other.
```cpp
std::mutex _mtx; // For synchronizing STI & LTI funds update
std::mutex AFMutex; // For AF fetching and update
```

- holds signals for when attention focus and attention value change.
- there is a type of signal emitted when atoms cross in and out of attention focus.
```cpp
/* Attention Value changed */
typedef SigSlot<const Handle&,
                const AttentionValuePtr&,
                const AttentionValuePtr&> AVCHSigl;

/* Attentional Focus changed */
typedef SigSlot<const Handle&,
                const AttentionValuePtr&,
                const AttentionValuePtr&> AFCHSigl;


AFCHSigl _AddAFSignal;
AFCHSigl _RemoveAFSignal;

// SIGNAL EMITTED WHEN AN ATOM CROSSES IN OR OUT OF AF
```
- Fund parameters
```cpp
AttentionValue::sti_t fundsSTI;
AttentionValue::lti_t fundsLTI;

AttentionValue::sti_t startingFundsSTI;
AttentionValue::lti_t startingFundsLTI;

AttentionValue::sti_t stiFundsBuffer;
AttentionValue::lti_t ltiFundsBuffer;

AttentionValue::sti_t targetSTI;
AttentionValue::lti_t targetLTI;

AttentionValue::sti_t STIAtomWage;
AttentionValue::lti_t LTIAtomWage;
```
- Maximum attention focus size: the maximum number of atoms that can be kept in  attentional focus.
```cpp
unsigned int maxAFSize;
```
- ImportanceIndex class: Implements an index with additional routines needed for managing short-term importance.  This index is thread-safe. 
- The importanceIndex class extends the functionalities of AtomBins by adding logic for managing and retrieving atoms based on sti values.
- core purpose: 
	- Indexes atoms into bins based on their importance (sti values).
	- provides utilities to update and retrieve atoms based on their importance.
	- tracks and manages short-term importance values (max and min sti values).
- its friendship with the ecan::StochasticDiffusionAmountCalculator class allows the later to access its private members and functions.
```cpp
/**
 * Implements an index with additional routines needed for managing
 * short-term importance.  This index is thread-safe.
 */
using HandleSTIPair = std::pair<Handle, AttentionValue::sti_t>;

class ImportanceIndex
{
    // Needs to access importanceBin
    friend class ecan::StochasticDiffusionAmountCalculator;
private:
    mutable std::mutex _mtx;

    AtomBins _index;

    /// Running average min and max STI.
    opencog::recent_val<AttentionValue::sti_t> _maxSTI;
    opencog::recent_val<AttentionValue::sti_t> _minSTI;

    /**
     * This method returns which importance bin an atom with the given
     * STI should be placed.
     *
     * @param Importance value to be mapped.
     * @return The importance bin which an atom of the given importance
     * should be placed.
     */
    static size_t importanceBin(AttentionValue::sti_t);

public:
    ImportanceIndex();
    void removeAtom(const Handle&);

    void update(void);

    /**
     * Get the maximum STI observed.
     *
     * @param average If true, return an exponentially decaying
     *        average of maximum STI, otherwise return the actual
     *        maximum.
     * @return Maximum STI
     */
    AttentionValue::sti_t getMaxSTI(bool average=true) const;

    /**
     * Get the minimum STI observed.
     *
     * @param average If true, return an exponentially decaying
     *        average of minimum STI, otherwise return the actual
     *        minimum.
     * @return Minimum STI
     */
    AttentionValue::sti_t getMinSTI(bool average=true) const;

    /**
     * Updates the importance index for the given atom.
     */
    void updateImportance(const Handle&,
                          const AttentionValuePtr& oldav,
                          const AttentionValuePtr& newav);

    /**
     * Returns the set of atoms within the given importance range.
     *
     * @param Importance range lower bound (inclusive).
     * @param Importance range upper bound (inclusive).
     * @return The set of atoms within the given importance range.
     */
    UnorderedHandleSet getHandleSet(AttentionValue::sti_t lowerBound,
                                    AttentionValue::sti_t upperBound =
                                         AttentionValue::MAXSTI) const;

    // Get the content of an ImportanceBin at index i.
    template <typename OutputIterator> OutputIterator
    getContent(size_t i,OutputIterator out) const
    {
        return _index.getContent(i,out);
    }

    Handle getRandomAtom(void) const;

    /**
     * Get the highest bin which contains Atoms
     */
    UnorderedHandleSet getMaxBinContents();

    /**
     * Get the lowest bin which contains Atoms
     */
    UnorderedHandleSet getMinBinContents();
    
    size_t bin_size(void) const;
    
    /**
     * Get the size of the bin at the given index.
     */
    size_t size(int) const;
};
```
- The signal emitted when av changes:
```cpp
/** Signal emitted when the AV changes. */
AVCHSigl _AVChangedSignal;
```
- change_vlti, atomspace, remove_atom_from_bank
```cpp
AtomSpace* _as;

void change_vlti(const Handle&, int);
void remove_atom_from_bank(const AtomPtr& atom);
```

all the information listed above are private to the class.

#### the public ones
```cpp
/**
 * Provide ability for others to find out about atoms that cross in or
 * out of the AttentionalFocus
 */
AFCHSigl& AddAFSignal() { return _AddAFSignal; }
AFCHSigl& RemoveAFSignal() { return _RemoveAFSignal; }

/** Provide ability for others to find out about AV changes */
AVCHSigl& getAVChangedSignal() { return _AVChangedSignal; }
```

- setters and getters for maxAFSize, getters for min and max sti values in attention focus
```cpp
AttentionValue::sti_t get_af_max_sti(void) const
{
	if (attentionalFocus.rbegin() != attentionalFocus.rend())
		return (attentionalFocus.rbegin()->second)->getSTI();
	else
		return 0;
}

AttentionValue::sti_t get_af_min_sti(void) const
{
	if (attentionalFocus.rbegin() != attentionalFocus.rend())
		return (attentionalFocus.begin()->second)->getSTI();
	else
		return 0;
}

void set_af_size(int size) {
	maxAFSize = size;
}

int get_af_size(void) {
	return maxAFSize;
}
```
- change attention value of atoms: set, inc(lti, vlti), dec(lti, vlti), change_av(update existing av with new instance)
```cpp
void change_av(const Handle&, const AttentionValuePtr& new_av);
void set_sti(const Handle&, AttentionValue::sti_t);
void set_lti(const Handle&, AttentionValue::lti_t);
void inc_vlti(const Handle& h) { change_vlti(h, +1); }
void dec_vlti(const Handle& h) { change_vlti(h, -1); }
```

```cpp
/**
 * Stimulate an atom.
 *
 * @warning Should only be used by attention allocation system.
 * @param  h Handle to be stimulated
 * @param stimulus stimulus amount
 */
void stimulate(const Handle&, double stimulus);
```

```cpp
/**
 * Get the total amount of STI in the AttentionBank, sum of
 * STI across all atoms.
 *
 * @return total STI in the AttentionBank
 */
AttentionValue::sti_t getTotalSTI() const {
	return startingFundsSTI - (AttentionValue::sti_t)fundsSTI;
}

/**
 * Get the total amount of LTI in the AttentionBank, sum of
 * all LTI across atoms.
 *
 * @return total LTI in the AttentionBank
 */
AttentionValue::lti_t getTotalLTI() const {
	return startingFundsLTI - (AttentionValue::lti_t)fundsLTI;
}

/**
 * Get the STI funds available in the AttentionBank pool.
 *
 * @return STI funds available
 */
AttentionValue::sti_t getSTIFunds() const { return fundsSTI; }

/**
 * Get the LTI funds available in the AttentionBank pool.
 *
 * @return LTI funds available
 */
AttentionValue::lti_t getLTIFunds() const { return fundsLTI; }
```

- Buffers
```cpp
AttentionValue::sti_t getSTIFundsBuffer(){ return stiFundsBuffer;}

AttentionValue::lti_t getLTIFundsBuffer(){ return ltiFundsBuffer;}
```

- Wage calculations
```cpp
AttentionValue::sti_t calculateSTIWage(void);

AttentionValue::lti_t calculateLTIWage(void);
```

- Fetching doubly normalized sti value of atom
```cpp

/**
 * Retrieve the doubly normalised Short-Term Importance between -1..1
 * for a given AttentionValue. STI above and below threshold
 * normalised separately and linearly.
 *
 * @param h The attention value holder to get STI for
 * @param average Should the recent average max/min STI be used,
 *        or the exact min/max?
 * @param clip Should the returned value be clipped to -1..1?
 *        Outside this range can be return if average=true
 * @return normalised STI between -1..1
 */
double getNormalisedSTI(AttentionValuePtr, bool average, bool clip) const;

/**
 * @see getNormalisedSTI()
 */
double getNormalisedSTI(AttentionValuePtr) const;
```
- Fetching linearly normalized sti value of atom
```cpp

/**
 * Retrieve the linearly normalised Short-Term Importance between 0..1
 * for a given AttentionValue.
 *
 * @param h The attention value holder to get STI for
 * @param average Should the recent average max/min STI be used,
 *        or the exact min/max?
 * @param clip Should the returned value be clipped to 0..1?
 *        Outside this range can be return if average=true
 * @return normalised STI between 0..1
 */
double getNormalisedZeroToOneSTI(AttentionValuePtr, bool average, bool clip) const;

bool atom_is_in_AF(const Handle&);
```
- checker if atom is in AF and a function that returns atoms in the attention focus
```cpp
bool atom_is_in_AF(const Handle&);

/**
 * Gets the set of all handles in the Attentional Focus
 *
 * @return The set of all atoms in the Attentional Focus
 * @note: This method utilizes the ImportanceIndex
 */
template <typename OutputIterator> OutputIterator
get_handle_set_in_attentional_focus(OutputIterator result)
{
	 std::lock_guard<std::mutex> lock(AFMutex);
	 for (const auto& p : attentionalFocus) {
		 *result++ = p.first;
	 }
	 return result;
}
```

- importance index and random atom from outside AF
```cpp
/// Return the index itself, giving direct access to it.
ImportanceIndex& getImportance()
{
	return _importanceIndex;
}

/// Return a random atom drawn from outside the AF.
Handle getRandomAtomNotInAF(void);
```
- Importance Index related methods
```cpp
AttentionValue::sti_t getMinSTI(bool average=true) const
{
	return _importanceIndex.getMinSTI(average);
}

AttentionValue::sti_t getMaxSTI(bool average=true) const
{
	return _importanceIndex.getMaxSTI(average);
}

UnorderedHandleSet getHandlesByAV(AttentionValue::sti_t lowerBound,
			  AttentionValue::sti_t upperBound = AttentionValue::MAXSTI) const
{
	return _importanceIndex.getHandleSet(lowerBound, upperBound);
}
/** Same as above, different API */
template <typename OutputIterator> OutputIterator
get_handles_by_AV(OutputIterator result,
				  AttentionValue::sti_t lowerBound,
				  AttentionValue::sti_t upperBound = AttentionValue::MAXSTI) const
{
	UnorderedHandleSet hs = getHandlesByAV(lowerBound, upperBound);
	return std::copy(hs.begin(), hs.end(), result);
}

```
- atomspace
```cpp
/* Singleton instance (for now) */
AttentionBank& attentionbank(AtomSpace*);
```

#### Atom Bins
- Implements a **bin classifier**: a mechanism for categorizing or storing items into different 'bins' based on some criteria. It is simply a bucket containing items sharing certain characteristics.
- bin or HandleSet: represents a set of unique handles. used to store items for fast lookups and no duplicates.
- idx or HandleSetSeq:  a vector of bin aka HandleSet representing a **sequence of bins**. Each index corresponds to a specific bin.
- using this bin logic getContent and getContentIf are implemented to get a Handle from an index. getContent finds a Handle from a bin and returns a copy of it where are getContentIf returns the desired output based on user defined conditions i.e predicates.
```cpp

#include <opencog/atoms/base/Handle.h> ////! a sequence of sets of handles.
// typedef std::vector<HandleSet> HandleSetSeq;


namespace opencog
{
/** \addtogroup grp_atomspace
 *  @{
 */

/**
 * Implements a bin classifier. what's a bin?
 */
class AtomBins
{
    private:
        mutable std::mutex _mtx;
        HandleSetSeq _idx; //vector of HandleSet

    public:
        AtomBins(size_t sz)
        {
            _idx.resize(sz);
        }

        void insert(size_t i, const Handle& a)
        {
            std::lock_guard<std::mutex> lck(_mtx);
            _idx.at(i).insert(a);
        }

        void remove(size_t i, const Handle& a)
        {
            std::lock_guard<std::mutex> lck(_mtx);
            _idx.at(i).erase(a);
        }

        size_t size(size_t i) const
        {
            std::lock_guard<std::mutex> lck(_mtx);
            return _idx.at(i).size();
        }

        Handle getRandomAtom(void) const;

        size_t size() const;

        template <typename OutputIterator> OutputIterator
        getContent(size_t i, OutputIterator out) const
        {
            std::lock_guard<std::mutex> lck(_mtx);
            const HandleSet& s(_idx.at(i));
            return std::copy(s.begin(), s.end(), out);
        }

        template <typename OutputIterator> OutputIterator
        getContentIf(size_t i,
                    OutputIterator out,
                    std::function<bool(const Handle&)> pred) const
        {
            std::lock_guard<std::mutex> lck(_mtx);
            const HandleSet& s(_idx.at(i));
            return std::copy_if(s.begin(), s.end(), out, pred);
        }
};

/** @}*/
} //namespace opencog
#
```

- Type definitions for atoms, taken from atomspace.
```cpp
// type definitions taken from atomspace/opencog/atoms/base/Handle.h
typedef std::set<Handle> HandleSet;

//! a set of sets of handles.
typedef std::set<HandleSet> HandleSetSet;

//! a sequence of sets of handles.
typedef std::vector<HandleSet> HandleSetSeq;

//! a hash table. Usually has faster insertion.
typedef std::unordered_set<Handle> UnorderedHandleSet;

//! an ordered map from Handle to Handle
typedef std::map<Handle, Handle> HandleMap;

//! a hash table. Usually has faster insertion.
typedef std::unordered_map<Handle, Handle> UnorderedHandleMap;

//! an ordered map from Handle to HandleSet
typedef std::map<Handle, HandleSet> HandleMultimap;

//! an ordered map from Handle to HandleSeq
typedef std::map<Handle, HandleSeq> HandleSeqMap;

//! a sequence of ordered handle maps
typedef std::vector<HandleMap> HandleMapSeq;

//! a sequence of sequences of ordered handle maps
typedef std::vector<HandleMapSeq> HandleMapSeqSeq;

//! a set of ordered handle maps
typedef std::set<HandleMap> HandleMapSet;

//! a sequence of handle pairs
typedef std::vector<HandlePair> HandlePairSeq;

//! a map from handle to double
typedef Counter<Handle, double> HandleCounter;

//! a map from handle to unsigned
typedef Counter<Handle, unsigned> HandleUCounter;

// A map of variables to thier groundings.  Everyone working with
// groundings uses this type; changing the type here allows easy
// comparisons of performance for these two mapping styles.
// At this time (Dec 2019; gcc-8.3.0) there seems to be no difference
// in performance in the pattern matcher as a result of using the
// unordered aka std::_Hashtable variant vs the std::_Rb_tree variant.
// (as measured with the `guile -l nano-en.scm` benchmark.)
typedef HandleMap GroundingMap;
// typedef UnorderedHandleMap GroundingMap;
typedef std::vector<GroundingMap> GroundingMapSeq;
typedef std::vector<GroundingMapSeq> GroundingMapSeqSeq;

```
#### AVUtils
-  Handy utility functions to get attention value of atoms
```cpp
/**
 * Handy utilities to get the attention value of an atom.
 */
AttentionValuePtr get_av(const Handle&);
void set_av(AtomSpace*, const Handle&, const AttentionValuePtr&);

// the above two functions get defined in its corresponding .cc file
static inline AttentionValue::sti_t get_sti(const Handle& h)
{
    return get_av(h)->getSTI();
}

static inline AttentionValue::lti_t get_lti(const Handle& h)
{
    return get_av(h)->getLTI();
}

static inline AttentionValue::vlti_t get_vlti(const Handle& h)
{
    return get_av(h)->getVLTI();
}


```
#### AttentionSCM
- this is where helper scheme running adapter functions that deal with attention values and attentional focus in atomspace are. They make use of the scheme defined functions under the hood.
- a short explanation of the functions is found in the readme from the attentionbank folder.
```cpp
class AttentionSCM
{
	protected:
		static void* init_in_guile(void*);
		static void init_in_module(void*);
		void init(void);
	public:
		AttentionSCM(void);
		~AttentionSCM();

		ValuePtr get_av(const Handle&);
		Handle set_av(const Handle&, const ValuePtr&);
		Handle inc_vlti(const Handle&);
		Handle dec_vlti(const Handle&);

		Handle update_af(int);
		int af_size(void);
		int set_af_size(int);
		Handle stimulate(const Handle&, double);

		Handle af_bindlink(const Handle&);
};
}
```

#### AttentionFocusCB (Attention Focus Call back)
- Imports callback mixin defined in atomspace/opencog/query/TermMatchMixin from as folder.
- This is used to implement the matcher functionality to pattern match over the atomspace.
- The basic ideas of pattern matching is found inside the atomspace repo under opencog/query
```cpp
class AttentionalFocusCB: public TermMatchMixin
{
public:
	AttentionalFocusCB(AtomSpace*);

	// Only match nodes if they are in the attentional focus
	bool node_match(const Handle&, const Handle&);

	// Only match links if they are in the attentional focus
	bool link_match(const PatternTermPtr&, const Handle&);

	// Only get incoming sets that are in the attentional focus
	IncomingSet get_incoming_set(const Handle&, Type);
};

```
#### StochasticImportanceDiffusion
- This seems to be concerned with the distribution of importance on different atoms.
- This uses it's own DiffusionRecordBin to control diffusion of importance.
- The diffusion process spreads or reduces **short-term importance (STI)** over time for a set of items (called "handles"). Here's the simplest way to understand it:
1. **Handles are grouped into bins** based on their STI value. Higher STI values go into higher bins.
2. **STI decreases over time** (decays) like a "cooling-off" effect:
    - Each handle's STI is reduced by a percentage over time.
    - The reduction depends on how much time has passed and a **decay rate**.
3. Each time a handle's STI changes:
    - It may be moved to a different bin.
    - The system keeps track of how often and how quickly bins are updated.
```cpp
        /**
         * Diffusion accross the whole atomspace in one time step is expensive.
         * So we randomly choose atom and diffuse their STI values. However, this
         * method requires keeping records of each atom's last diffusion time which
         * again will be computationaly expensive for large atomspace. To overcome
         * this problem, we use a stochastic estimation of the average elpased time by
         * grouping atoms in to fixed number of bins and recording and updating the last
         * diffusion event time related to an arbitrary atom belonging to a particular
         * bin and a count of how many atoms have been updated so far in that
         * particular bin. Then the average elapsed time since last diffusion event
         * for an atom will be claculated as total in the bin divided by update rate
         * (count of diffused atoms divided by duration of time).
         */

class ImportanceIndex;
class Handle;

struct DiffusionRecordBin {
	unsigned int count = 0; // Number of atoms diffused.
	unsigned int index = 0; // The index of this Bin in ImportanceIndex vec.
	unsigned int size = 0;  // The total size of this Bin in ImportanceIndex vec.
	float update_rate = 0;  // Estimated update rate for this bin.

	// Latest time of diffusion event in this bin.
	time_point<high_resolution_clock> last_update=high_resolution_clock::now(); 
};

class StochasticDiffusionAmountCalculator
{
	ImportanceIndex* _imidx;
	std::vector<DiffusionRecordBin> _bins; 

	size_t bin_index(const Handle&);
	size_t bin_size(unsigned int index);
	void update_bin(const Handle&);

public:
	StochasticDiffusionAmountCalculator(ImportanceIndex*);

	std::vector<DiffusionRecordBin> merge_bins(
			const std::vector<DiffusionRecordBin>& past,
			std::vector<DiffusionRecordBin>& recent,
			float bias);
	float diffused_value(const Handle& h, float decay_rate);
	float elapsed_time(const Handle& h);
};
```

### modules that are directly related regarding functionalities
#### Attention value and attentional focus change communication module:
- AFImplicator
- AttentionalFocusCB
- AttentionBank
#### Attention value distribution module:
- AtomBins
- ImportanceIndex
- StochasticImportanceDiffusion
#### Attention value allocation module:
- AttentionBank
- AttentionValue, AttentionValueOfLink
- ImportanceIndex
#### AttentionSCM 
* These are Cpp adapters that run scheme defined functions to interact with the atomspace to:
	* set and get attention values of atoms, increase or decrease individual values of attention values (sti, lti, vlti)
	* stimulate Handles.
	* update attentional focus size etc ... 
* Therefore this seems to be independent of the others.