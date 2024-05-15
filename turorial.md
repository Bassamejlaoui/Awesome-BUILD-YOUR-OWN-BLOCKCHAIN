# BUILD YOUR OWN BLOCKCHAIN: A PYTHON TUTORIAL
**Download the full Jupyter/iPython notebook from Github [here](https://github.com/mejbass/Awesome-BUILD-YOUR-OWN-BLOCKCHAIN/blob/main/BuildYourOwnBlockchain.ipynb)**

## Build Your Own Blockchain – The Basics
This tutorial will walk you through the basics of how to build a blockchain from scratch. Focusing on the details of a concrete example will provide a deeper understanding of the strengths and limitations of blockchains. For a higher-level overview, I’d recommend this excellent article from BitsOnBlocks.

## Transactions, Validation, and updating system state
At its core, a blockchain is a distributed database with a set of rules for verifying new additions to the database. We’ll start off by tracking the accounts of two imaginary people: Alice and Bob, who will trade virtual money with each other.

We’ll need to create a transaction pool of incoming transactions, validate those transactions, and make them into a block.

We’ll be using a [hash function](https://en.wikipedia.org/wiki/SHA-2) to create a ‘fingerprint’ for each of our transactions- this hash function links each of our blocks to each other. To make this easier to use, we’ll define a helper function to wrap the [python hash function](https://docs.python.org/2/library/hashlib.html) that we’re using.

In [1]:

```python
import hashlib
import json
import sys

def hashMe(msg=""):
    """
    Helper function that wraps the hashing algorithm.
    
    Args:
        msg (str or dict): The message to be hashed.
    
    Returns:
        str: The hashed message.
    """
    if type(msg) != str:
        msg = json.dumps(msg, sort_keys=True)  # If we don't sort keys, we can't guarantee repeatability!
        
    if sys.version_info.major == 2:
        return unicode(hashlib.sha256(msg).hexdigest(), 'utf-8')
    else:
        return hashlib.sha256(str(msg).encode('utf-8')).hexdigest()
```

Next, we want to create a function to generate exchanges between Alice and Bob. We’ll indicate withdrawals with negative numbers, and deposits with positive numbers. We’ll construct our transactions to always be between the two users of our system, and make sure that the deposit is the same magnitude as the withdrawal- i.e. that we’re neither creating nor destroying money

In [2]:

```python
import random

random.seed(0)

def makeTransaction(maxValue=3):
    """
    Creates a valid transaction between Alice and Bob.
    
    Args:
        maxValue (int): The maximum value for the transaction.
    
    Returns:
        dict: A dictionary representing the transaction between Alice and Bob.
    """
    # This will create valid transactions in the range of (1, maxValue)
    sign = int(random.getrandbits(1)) * 2 - 1   # This will randomly choose -1 or 1
    amount = random.randint(1, maxValue)
    alicePays = sign * amount
    bobPays = -1 * alicePays
    # By construction, this will always return transactions that respect the conservation of tokens.
    # However, note that we have not done anything to check whether these overdraft an account
    return {'Alice': alicePays, 'Bob': bobPays}
```

Now let’s create a large set of transactions, then chunk them into blocks.

In [3]:

```python
txnBuffer = [makeTransaction() for i in range(30)]
```

Next step: making our very own blocks! We’ll take the first k transactions from the transaction buffer, and turn them into a block. Before we do that, we need to define a method for checking the valididty of the transactions we’ve pulled into the block.

For bitcoin, the validation function checks that the input values are valid unspent transaction outputs (UTXOs), that the outputs of the transaction are no greater than the input, and that the keys used for the signatures are valid. In Ethereum, the validation function checks that the smart contracts were faithfully executed and respect gas limits.

No worries, though- we don’t have to build a system that complicated. We’ll define our own, very simple set of rules which make sense for a basic token system:

**The sum of deposits and withdrawals must be 0 (tokens are neither created nor destroyed)**
**A user’s account must have sufficient funds to cover any withdrawals**

If either of these conditions are violated, we’ll reject the transaction.

In [4]:

```python
def updateState(txn, state):
    """
    Updates the state based on the transaction.

    Inputs:
        txn (dict): Transaction dictionary keyed with account names, holding numeric values for transfer amount.
        state (dict): State dictionary keyed with account names, holding numeric values for account balance.

    Returns:
        dict: Updated state with additional users added if necessary.
    """
    # If the transaction is valid, then update the state
    state = state.copy()  # As dictionaries are mutable, let's avoid any confusion by creating a working copy of the data.
    for key in txn:
        if key in state.keys():
            state[key] += txn[key]
        else:
            state[key] = txn[key]
    return state
```
