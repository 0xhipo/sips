|   SIP-Number |  |
|         ---: | :--- |
|        Title | ZK Set Accumulator for Efficient On-Chain Accounting |
|  Description | A set accumulator using Merkle Trie with O(log N) on-chain verification, storing data off-chain on Walrus Protocol, optimizing on-chain accounting operations. |
|       Author | 0xhipo and Lotus Finance |
|       Editor |  |
|         Type | Standard |
|     Category | Framework |
|      Created | 2025-02-28 |
| Comments-URI |  |
|       Status |  |
|     Requires |  |

# Abstract

This proposal introduces a set accumulator for the Sui blockchain to enable efficient on-chain verification of membership and non-membership in large datasets. The set is represented as a Merkle Trie, with the full trie stored off-chain on Walrus Protocol and the Merkle root maintained on-chain. On-chain (root) state can only be modified with valid and verified membership or non-membership proof provided. Smart contracts verify proofs in O(log N) time, O(1) space. A reference Python implementation for proof generation and verification logic is also provided.

# Motivation

Handling large datasets on-chain is costly and inefficient, especially in applications requiring frequent updates or complex computations. This set accumulator addresses this challenge by offloading data storage to Walrus Protocol and optimizing on-chain verification. Specific use cases include:

- **Game**: User interactions in games are often numerous (e.g., millions of actions in a multiplayer environment). Storing and processing all interactions on-chain is impractical. The accumulator allows computation of functions (e.g., total points, unique actions) over the set of interactions, with proofs verified efficiently on-chain.
  
- **DeFi**: Trading data in decentralized finance grows with assets, users, and accumulate by time. Accounting for every trade on-chain is infeasible due to gas costs and storage limits. The accumulator enables functions (e.g., total volume, trade validation) over large trade datasets, verified with minimal on-chain time / space complexity.
  
- **Oracle**: Oracles often handle bulk data (e.g., price feeds, weather records) that cannot be stored on-chain. By storing data on Walrus Protocol and computing a Merkle Trie, oracles can post membership or non-membership proofs on-chain, allowing smart contracts to verify data integrity without storing the full dataset.

These use cases demonstrate the need for a scalable solution that balances off-chain storage with efficient on-chain verification.

Consider a more specific scenario, as in Lotus Finance, we want to support auditing of trade data for a vault, and define a metric upon set of trades. The trade data is stored as [`Fill`](https://github.com/MystenLabs/deepbookv3/blob/f0384222741e0f12e12dd3aec34f73da694a6480/packages/deepbook/sources/book/fill.move#L13) in Deepbook. For each Vault we want to execute a transaction for each trade it makes, adds to a market maker score for doing that traade. It's easy for the contract to verify the trade exists, however, to make sure the trades are not accounted duplicated, maintaining an accounted trade set would require O(N) in both time and space complexity. As time goes and mass adoption of traders, the number of trades could be in the order of millions on which scale we don't want to perform O(N) operations on-chain. Thus we need a data structure to store set off-chain, only send a proof of that trade is non-membership of the set to the contract, to prevent double accounting.

# Specification

- **Element**: String (vector<u8>) representing ID of a set member.

- **Data Structure**: The set is represented as a Merkle Trie, where each node’s hash is computed from its children’s hashes using SHA-256. Leaf nodes represent set elements.
  
- **Off-Chain Storage**: The full Merkle Trie is maintained on Walrus Protocol, a decentralized storage solution, allowing scalable management of large datasets.
  
- **On-Chain Storage**: The smart contract stores only the Merkle root, a 32-byte hash representing the entire set.

- **Proof Generation**: For member proof an audit path is generated for each element, containing the selected child and full children mapping at each level until reaches that specific element. For non-membership proof the path is generated until the divergence point.

- **On-Chain State Modification**: The Merkle root can only be updated with valid proofs of membership or non-membership, ensuring data integrity and preventing unauthorized modifications.

## Smart Contract Functions (in Move)

- `update_root(new_root: vector<u8>)`: Updates the stored Merkle root after off-chain trie modifications, restricted to the contract owner or authorized entity. Should verify membership or non-membership before calling this.
  
- `verify_membership_proof(element: vector<u8>, proof: vector<vector<u8>>, root: vector<u8>): bool`: Verifies a membership proof by recomputing the Merkle root from the element and proof path, comparing it to the stored root.
  
- `verify_non_membership_proof(element: vector<u8>, proof: vector<vector<u8>>, root: vector<u8>): bool`: Verifies a non-membership proof similarly, ensuring the element’s absence in the set.

## Reference Implementation

The provided Python code implements the Merkle Trie, including `insert`, `build_merkle_root`, `generate_membership_proof`, `generate_non_membership_proof`, `verify_membership_proof`, and `verify_non_membership_proof`. The Move implementation mirrors this logic, using SHA-256 for hashing.

- On-chain part: `insert`, `verify_membership_proof`, `verify_non_membership_proof`,
- Off-chain part: `build_merkle_root`, `generate_membership_proof`, `generate_non_membership_proof`.

## Complexity

Proof verification is O(log N), where N is the number of elements, due to the logarithmic depth of the trie.

# Rationale

Merkle Tries are chosen for their proven efficiency in proof-based systems (e.g., Ethereum state trie), offering O(log N) verification. Walrus Protocol provides scalable, decentralized off-chain storage, complementing Sui’s high-throughput design. 

# Backwards Compatibility

This is a new feature with no impact on existing contracts or systems, requiring no migration. More discussions 
are needed on whether we should use Sui Move's ZK api groth16 in this implementation.

# Security Considerations

- **Data Integrity**: The Merkle root ensures that any changes to the off-chain trie are tamper-evident. This means that any modification to the trie must result in a new Merkle root that matches the one stored on-chain, ensuring the integrity of the data.
  
- **Off-Chain Management**: Assumes secure trie updates on Walrus Protocol, with root updates controlled by a trusted entity or governance mechanism.
  
- **Proof Verification**: SHA-256 is collision-resistant, ensuring proof reliability.

# Optional Sections

## Discussion

While the proposal avoids Sui’s `verify_groth16` API to focus on simplicity and efficiency, future enhancements could use Sui Move's ZK api groth16 for better performance and privacy preserving. 

## Test Cases

- Verify membership proof for an element in a 1,000-element trie, and examine the time complexity.
- Verify non-membership proof for an absent element, and examine the time complexity.
- Test root update with a modified trie. Should fail.

## Reference Implementation

```python
import hashlib

class MerkleTrie:
    def __init__(self):
        self.root = {}

    def insert(self, element):
        current = self.root
        for char in element:
            if char not in current:
                current[char] = {}
            current = current[char]
        # Mark the leaf with the special key '$'
        current['$'] = element

    def _compute_hash(self, node):
        if '$' in node and len(node) == 1:
            node_hash = hashlib.sha256(node['$'].encode()).hexdigest()
            node['__hash__'] = node_hash
            return node_hash

        child_hashes = []
        for k in sorted(node.keys()):
            if k == '__hash__':
                continue
            if k == '$':
                child_hashes.append(hashlib.sha256(node['$'].encode()).hexdigest())
            else:
                child_hashes.append(self._compute_hash(node[k]))
        combined = ''.join(child_hashes)
        node_hash = hashlib.sha256(combined.encode()).hexdigest()
        node['__hash__'] = node_hash
        return node_hash

    def build_merkle_root(self):
        return self._compute_hash(self.root)

    def get_merkle_root(self):
        return self.root.get('__hash__', '')

    def get_auditable_membership_proof(self, element):
        """
        Generates an auditable membership proof along the path of the element.
        For each level that exists in the trie, record:
          (selected_child, full_children_mapping)
        where full_children_mapping includes the hash of every child at that node.
        """
        proof = []
        current = self.root
        for char in element:
            if char not in current:
                break
            full_mapping = {}
            for k, v in current.items():
                if k == '__hash__':
                    continue
                if k == '$':
                    full_mapping['$'] = hashlib.sha256(current['$'].encode()).hexdigest()
                else:
                    full_mapping[k] = v.get('__hash__', '')
            proof.append((char, full_mapping))
            current = current[char]
        return proof

    def get_non_membership_proof(self, element):
        """
        Generates a non-membership proof by traversing the trie using the element
        string until at some level the character is missing.
        The returned proof is a dict with:
          "path": auditable proof (list of (selected_char, full_children_mapping)) 
                  from the root down to the divergence point.
          "divergence": a dict containing:
              - "missing": the diverging character (from the queried element)
              - "children": the full children mapping at that divergence node.
        This information is sufficient to recompute the node hash at divergence and upward.
        If the element exists in the trie entirely, returns None.
        """
        current = self.root
        auditable_path = []
        i = 0
        while i < len(element):
            full_mapping = {}
            for k, v in current.items():
                if k == '__hash__':
                    continue
                if k == '$':
                    full_mapping['$'] = hashlib.sha256(current['$'].encode()).hexdigest()
                else:
                    full_mapping[k] = v.get('__hash__', '')
            if element[i] not in current:
                # Divergence found.
                divergence = {"missing": element[i], "children": full_mapping}
                return {"path": auditable_path, "divergence": divergence}
            # Otherwise, add the current level to the auditable path.
            auditable_path.append((element[i], full_mapping))
            current = current[element[i]]
            i += 1
        # If the entire element is traversed, then it exists in the trie.
        return None

def compute_auditable_hash_from_path(leaf_hash, auditable_path):
    """
    Given a starting leaf hash and an auditable path (a list of tuples)
    rebuild the hash upward in the Merkle trie.
    """
    current_hash = leaf_hash
    for selected, full_mapping in reversed(auditable_path):
        mapping = full_mapping.copy()
        mapping[selected] = current_hash
        sorted_keys = sorted(mapping.keys())
        child_hashes = [mapping[k] for k in sorted_keys]
        combined = ''.join(child_hashes)
        current_hash = hashlib.sha256(combined.encode()).hexdigest()
    return current_hash

def verify_membership_proof(element, proof, root_hash):
    """
    Verifies a membership proof by reconstructing the Merkle root starting
    from the leaf (hash of element) upward using the auditable proof.
    """
    if not proof or not root_hash:
        return False
    leaf_hash = hashlib.sha256(element.encode()).hexdigest()
    computed_root = compute_auditable_hash_from_path(leaf_hash, proof)
    return computed_root == root_hash

def verify_non_membership_proof(element, proof, root_hash):
    """
    Verifies a non-membership proof. The proof is structured as a dict:
      {
        "path": auditable proof (list of (selected_char, full_children_mapping)) 
                from the root down to the divergence point,
        "divergence": {
              "missing": <diverging character from element>,
              "children": <full children mapping at divergence node>
        }
      }
    Verification requires that:
      1. The auditable path exactly matches the element's characters up to divergence.
      2. The divergence occurs at the first position where the element's character is missing 
         from the provided children mapping.
      3. The divergence children mapping does not include the diverging character.
      4. Rebuild the divergence node hash from the children mapping and then reconstruct 
         the root hash upward using the auditable path. The final hash must match root_hash.
    """
    if not proof or "divergence" not in proof:
        return False

    path = proof.get("path", [])
    divergence = proof["divergence"]

    # 1. Check that the auditable path exactly matches the element's characters.
    if len(path) >= len(element):
        # If path length equals or exceeds element length, then element should exist.
        return False

    for i, (selected_char, full_mapping) in enumerate(path):
        if element[i] != selected_char:
            # The auditable path does not follow the queried element.
            return False

    # 2. The divergence should occur at the next character.
    divergence_index = len(path)
    if divergence_index >= len(element):
        # No divergence can be found if element is shorter than or equal to path.
        return False
    diverging_char = element[divergence_index]
    if divergence.get("missing") != diverging_char:
        # The divergence record does not indicate the expected missing character.
        return False

    # 3. Ensure the divergence children do NOT contain the diverging character.
    if diverging_char in divergence.get("children", {}):
        return False

    # 4. Recompute the hash at the divergence node using the provided children mapping.
    children_mapping = divergence.get("children", {})
    sorted_keys = sorted(children_mapping.keys())
    child_hashes = [children_mapping[k] for k in sorted_keys]
    divergence_hash = hashlib.sha256("".join(child_hashes).encode()).hexdigest()

    # Rebuild the Merkle root from the divergence node upward using the auditable path.
    # This uses the helper function defined elsewhere:
    computed_root = compute_auditable_hash_from_path(divergence_hash, path)
    return computed_root == root_hash

def pretty_print_trie(node, indent=0):
    """
    Recursively prints the trie structure along with stored hash values.
    """
    prefix = " " * indent
    for k, v in node.items():
        if k == '__hash__':
            print(f"{prefix}__hash__: {v}")
        elif k == '$':
            print(f"{prefix}$: {v}")
        else:
            print(f"{prefix}{k}:")
            if isinstance(v, dict):
                pretty_print_trie(v, indent + 2)
            else:
                print(f"{' '*(indent+2)}{v}")

if __name__ == "__main__":
    # Sample usage.
    elements = ["abcd", "abce", "abcf"]
    trie = MerkleTrie()
    for el in elements:
        trie.insert(el)

    root = trie.build_merkle_root()
    print("Trie structure:")
    pretty_print_trie(trie.root)
    print("\nMerkle Root:", root)

    # Test membership proof.
    test_element = "abce"
    mem_proof = trie.get_auditable_membership_proof(test_element)
    print("\nMembership Proof for", test_element, ":\n", mem_proof)
    print("Verify Membership Proof:", verify_membership_proof(test_element, mem_proof, root))

    # Test non-membership proof.
    non_member = "abbd"
    non_mem_proof = trie.get_non_membership_proof(non_member)
    print("\nNon-Membership Proof for", non_member, ":\n", non_mem_proof)
    # Verify non-membership only if proof is generated
    if non_mem_proof is not None:
        print("Verify Non-Membership Proof:", verify_non_membership_proof(non_member, non_mem_proof, root))
    else:
        print("Non-Membership Proof could not be generated (element exists).")
```

# Copyright

MIT License, Copyright (c) 2025 0xhipo and Lotus Finance

# Implementation Notes

- **Off-Chain Process**: Users or oracles maintain the trie on Walrus Protocol, compute proofs, and submit them with elements for on-chain verification.
  
- **On-Chain Process**: The smart contract verifies proofs by recomputing the Merkle root from the proof path, ensuring it matches the stored root.
  
- **Scalability**: Storing only the root on-chain minimizes storage costs, while Walrus Protocol handles the full dataset.
