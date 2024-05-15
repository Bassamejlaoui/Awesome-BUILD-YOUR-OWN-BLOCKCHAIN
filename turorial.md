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

In [5]:

```python
def isValidTxn(txn, state):
    """
    Checks if the transaction is valid.

    Args:
        txn (dict): Transaction dictionary keyed by account names.
        state (dict): State dictionary keyed by account names, holding numeric values for account balance.

    Returns:
        bool: True if the transaction is valid, False otherwise.
    """
    # Assume that the transaction is a dictionary keyed by account names

    # Check that the sum of the deposits and withdrawals is 0
    if sum(txn.values()) != 0:
        return False
    
    # Check that the transaction does not cause an overdraft
    for key in txn.keys():
        if key in state.keys(): 
            acctBalance = state[key]
        else:
            acctBalance = 0
        if (acctBalance + txn[key]) < 0:
            return False
    
    return True
```

Here are a set of sample transactions, some of which are fraudulent- but we can now check their validity!

In [6]:

```python
state = {'Alice': 5, 'Bob': 5}

print(isValidTxn({'Alice': -3, 'Bob': 3}, state))  # Basic transaction- this works great!
print(isValidTxn({'Alice': -4, 'Bob': 3}, state))  # But we can't create or destroy tokens!
print(isValidTxn({'Alice': -6, 'Bob': 6}, state))  # We also can't overdraft our account.
print(isValidTxn({'Alice': -4, 'Bob': 2, 'Lisa': 2}, state))  # Creating new users is valid
print(isValidTxn({'Alice': -4, 'Bob': 3, 'Lisa': 2}, state))  # But the same rules still apply!
```
True

False

False

True

False


Each block contains a batch of transactions, a reference to the hash of the previous block (if block number is greater than 1), and a hash of its contents and the header

## Building the Blockchain: From Transactions to Blocks:

We’re ready to start making our blockchain! Right now, there’s nothing on the blockchain, but we can get things started by defining the ‘genesis block’ (the first block in the system). Because the genesis block isn’t linked to any prior block, it gets treated a bit differently, and we can arbitrarily set the system state. In our case, we’ll create accounts for our two users (Alice and Bob) and give them 50 coins each.

In [7]:

```python
state = {'Alice': 50, 'Bob': 50}  # Define the initial state
genesisBlockTxns = [state]
genesisBlockContents = {'blockNumber': 0, 'parentHash': None, 'txnCount': 1, 'txns': genesisBlockTxns}
genesisHash = hashMe(genesisBlockContents)
genesisBlock = {'hash': genesisHash, 'contents': genesisBlockContents}
genesisBlockStr = json.dumps(genesisBlock, sort_keys=True)
```

Great! This becomes the first element from which everything else will be linked.

In [8]:

```python
chain = [genesisBlock]
```

For each block, we want to collect a set of transactions, create a header, hash it, and add it to the chain

In [9]:

```python
def makeBlock(txns, chain):
    """
    Creates a new block.

    Args:
        txns (list): List of transactions.
        chain (list): List of blocks representing the blockchain.

    Returns:
        dict: A new block.
    """
    parentBlock = chain[-1]
    parentHash = parentBlock['hash']
    blockNumber = parentBlock['contents']['blockNumber'] + 1
    txnCount = len(txns)
    blockContents = {'blockNumber': blockNumber, 'parentHash': parentHash,
                     'txnCount': len(txns), 'txns': txns}
    blockHash = hashMe(blockContents)
    block = {'hash': blockHash, 'contents': blockContents}
    
    return block
```

Let’s use this to process our transaction buffer into a set of blocks:

In [10]:

```python
blockSizeLimit = 5  # Arbitrary number of transactions per block- 
                    #  this is chosen by the block miner, and can vary between blocks!

while len(txnBuffer) > 0:
    bufferStartSize = len(txnBuffer)
    
    ## Gather a set of valid transactions for inclusion
    txnList = []
    while (len(txnBuffer) > 0) & (len(txnList) < blockSizeLimit):
        newTxn = txnBuffer.pop()
        validTxn = isValidTxn(newTxn, state) # This will return False if txn is invalid
        
        if validTxn:           # If we got a valid state, not 'False'
            txnList.append(newTxn)
            state = updateState(newTxn, state)
        else:
            print("ignored transaction")
            sys.stdout.flush()
            continue  # This was an invalid transaction; ignore it and move on
        
    ## Make a block
    myBlock = makeBlock(txnList, chain)
    chain.append(myBlock)
```

In [11]:

```python
chain[0]
```

Out[11]:
{'contents': {'blockNumber': 0,
  'parentHash': None,
  'txnCount': 1,
  'txns': [{'Alice': 50, 'Bob': 50}]},
 'hash': '7c88a4312054f89a2b73b04989cd9b9e1ae437e1048f89fbb4e18a08479de507'}


In [12]:

```python
chain[1]
```

Out[12]:
{'contents': {'blockNumber': 1,
  'parentHash': '7c88a4312054f89a2b73b04989cd9b9e1ae437e1048f89fbb4e18a08479de507',
  'txnCount': 5,
  'txns': [{'Alice': 3, 'Bob': -3},
   {'Alice': -1, 'Bob': 1},
   {'Alice': 3, 'Bob': -3},
   {'Alice': -2, 'Bob': 2},
   {'Alice': 3, 'Bob': -3}]},
 'hash': '7a91fc8206c5351293fd11200b33b7192e87fad6545504068a51aba868bc6f72'}
As expected, the genesis block includes an invalid transaction which initiates account balances (creating tokens out of thin air). The hash of the parent block is referenced in the child block, which contains a set of new transactions which affect system state. We can now see the state of the system, updated to include the transactions:

In [13]:

```python
state
```

Out[13]:


{'Alice': 72, 'Bob': 28}

