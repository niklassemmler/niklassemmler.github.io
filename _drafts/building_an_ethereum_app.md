General:
- Mist wallet for actually storing ether
- Go-ethereum: https://github.com/ethereum/go-ethereum

Three protocols seem to be available:
- Vyper: https://github.com/ethereum/vyper
- Solidity: https://github.com/ethereum/solidity
- Bamboo: https://github.com/pirapira/bamboo

(from:) https://ethereum.stackexchange.com/questions/350/what-are-the-contract-languages

We use solidity, because it has the best traction and should have the most
extensive documentation.

https://solidity.readthedocs.io/en/develop/types.html

An online compilation tool can be found here:

https://remix.ethereum.org/#optimize=false&version=soljson-v0.4.19+commit.c4cbbb05.js

From my memory I recall, that we can deploy our app on a test net. Back a year
ago we still had to setup an ethereum node for this. Did this become simpler?

Also on the Java script side above, "Swarm" is mentioned. What is this good for?
Seems to be a distributed storage component for external content (e.g. websites
for dapps.) For now we do not need this.

Tutorial on Solidity itself:
- https://www.ethereum.org/greeter

Tutorial on setting up private network
- https://omarmetwally.blog/2017/07/25/how-to-create-a-private-ethereum-network/

For the future parity might be a good replacement of geth
- https://github.com/paritytech/parity

Main question: Do we need to install geth for developing contracts?
Answer: Yes, otherwise we cannot deploy it.
