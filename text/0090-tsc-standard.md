- **TEP**: [90](https://github.com/ton-blockchain/TEPs/pull/90)
- **title**: Token Signature Contract
- **status**: DRAFT
- **type**: Contract Interface
- **authors**: [Serge Polyanskikh](https://github.com/sergey-msu), [Roman H.](https://github.com/usert0n), ...
- **created**: 10.29.2022
- **replaces**: -
- **replaced by**: -

# Summary

Token Signature Contract (TSC) - is a special type of contract which acts as an agreement between the NFT owner and the  signatory. Leaving a signature on an NFT object serves as a validation by the third party. The signature belongs to its specific NFT and cannot be forwarded to anyone, as it is being a special type of SBT ([TEP 89](https://github.com/ton-blockchain/TEPs/blob/master/text/0085-sbt-standard.md)).

# Motivation

The concept of signing NFT objects is a demanded and natural tool, which has lots of potential applications. For example, an electronic document in the blockchain, that must be signed by interested parties, can be implemented as an NFT object. In this case, the NFT signing concept presented here is a fundamental base of electronic document management on TON blockchain. Another entertaining application has to do with looking at a signature as an autograph. In case a celebrity signs an NFT object, its collectible value will instantly increase, and will also serve as an additional anti-scam verification of the object.

# Guide

<p align="center">
 <img src="../assets/tsc-scheme.svg" alt="train_perf_fig"/>
    <br>
    <em>Fig. 1: General scheme of signing NFT objects.</em>
</p>

TSC is an agreement between an NFT object owner and a signatory. It must be released by some official signature authority that should be trusted by both the owner & the signatory. 

The general scheme of the solution is shown in Fig. 1. Provider here is an authority contract mentioned above that can have a basic NFT collection interface. It is responsible for issuing (mint) Signature contracts at the request of either the NFT owner (Owner) or a third party (Signatory). The Signature contract issued by the Provider belongs to the originally signed object without the right to transfer. Thus, the Signature contract is essentially a special kind of SBT token.

Approval of the signature by the signer can be carried out by a transaction to the signing contract with a special op code and some optional payload (e.g. the signatory's text message). This ensures that the signer's approved signature cannot be compromised by a phishing attack or otherwise.

Approval of the signature from the NFT owner side is more complicated. The transaction on the part of both, the owner of the NFT and the NFT itself, cannot serve as a confirmation of the signature, since before it is completed, the object can be transferred to another owner, which can create an unpleasant side effect. The solution may be to vote the NFT object, which is, to transfer it to the signature contract and immediately return it to the previous (initial) owner.

The described architecture for the implementation of NFT signing does not require an extension of the existing NFT [standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md) and can be implemented using already existing mechanics defined by the standard. Thus, once it is implemented, the signing scheme can be used for all previously issued NFTs whose contracts are compatible with the existing [standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md).

# Specification

The presence of Signature Provider authority in the general scheme is optional, and the details of its implementation may depend on specific business goals. Thus, it does not need a unique specification.

TSC storage TL-B schema:

```
storage#_
provider_address:MsgAddressInt owner_address:MsgAddressInt 
object_address:MsgAddressInt signatory_address:MsgAddressInt 
is_approved_by_owner:Bool is_approved_by_signatory:Bool 
signature_payload:(Maybe ^Cell) = Storage;
```

`provider_address` -  address of the signature provider authority.

`object_address` -  address of the NFT object.

`signatory_address` -  signatory address.

`is_approved_by_owner` - if true, TSC is approved by NFT owner.

`is_approved_by_signatory` - if true, TSC is approved by signatory.

`signature_payload` - arbitrary signature content (e.g. text message).

The TSC instance address depends only on a triplet: `provider_address`, `object_address`, `signatory_address`.

#### 1. `approve_by_signatory`

TL-B schema of inbound message:
```
approve_by_signatory#3eecd61c query_id:uint64 
signature_payload:(Maybe ^Cell) response_destination:MsgAddress 
response_amount:(VarUInteger 16) response_payload:(Either Cell ^Cell) = InternalMsgBody;
```
`query_id` -  arbitrary request number.

`signature_payload` -  arbitrary content to store in the Signature contract (e.g. text message).

`response_destination` - address where to send a response with confirmation of a successful signature approve by the signatory.

`response_amount` - amount of coins to be sent to the response address.

`response_payload` - optional custom data that should be sent to the response address.

**Should be rejected if:**

1. Message is not from `signatory_address` address.
2. There are not enough coins to process operation and send message to `response_destination` address.
3. The field `is_approved_by_signatory` is already set to True.

**Otherwise should do:**

1. Set `is_approved_by_signatory` field to True.
2. Set `signature_payload` field to incoming signature_payload cell.
3. If there is nonempty `response_destination` and nonzero `response_amount`, send `response_amount` of nanotons along with the `forward_payload` to this address with the following TL-B scheme:
```
notify_approve#a2761bc0 query_id:uint64 signatory_address:MsgAddress 
response_payload:(Either Cell ^Cell) = InternalMsgBody;
```
Here `query_id` should be equal with request's `query_id`. `response_payload` should be equal with request's `forward_payload`. `signatory_address` is address of the signatory. If `response_amount` is equal to zero, notification message should not be sent.

#### 2. `approve_by_owner`

TL-B schema of inbound message is exactly the same as in the [standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md) in the section describes sending message to new NFT owner address (Signature contract in this case) after transferring it:
```
ownership_assigned#05138d91 query_id:uint64 prev_owner:MsgAddress
forward_payload:(Either Cell ^Cell) = InternalMsgBody;
```
`query_id` -  arbitrary request number.

`prev_owner` - previous (real) owner of NFT object.

`forward_payload` - ignored by Signature contract.

**Should be rejected if:**

1. Message is not from `object_address`.

**Otherwise should do:**

1. Set `is_approved_by_owner` field to True.
2. Return the NFT object to its owner by the standard transfer ownership operation described in the [standard](https://github.com/ton-blockchain/TEPs/blob/master/text/0062-nft-standard.md). The TL-B scheme of outbound message should also coincide with the one from the standard:
```
transfer#5fcc3d14 query_id:uint64 new_owner:MsgAddress
response_destination:MsgAddress custom_payload:(Maybe ^Cell) 
forward_amount:(VarUInteger 16) forward_payload:(Either Cell ^Cell) = InternalMsgBody;
```
Here `query_id` is arbitrary request number. `new_owner` address must be equal to `prev_owner` one from the incoming message. The rest parameters may depend on the specific business problem being solved.

### Get-methods

1. `get_signature_data()` - returns signature-specific information: 
`(slice provider_address, slice object_address, slice signatory_address, int is_approved_by_owner, int is_approved_by_signatory, cell signature_payload)`.
Here `provider_address` is signature provider address (if exists), `object_address` is the address of the NFT object, `signatory_address` is the signatory address, `is_approved_by_owner` - if true, TSC is approved by NFT owner, `is_approved_by_signatory` - if true, TSC is approved by signatory, `signature_payload` is the arbitrary signature content (e.g. text message).

# Drawbacks
The bottleneck of this architecture is the signature approval from the NFT object owner's side. If it is implemented through voting described above, there will be a risk of freezing the NFT object on TSC in case of the incorrect transfer of it to the TSC (without or with incorrect forward transaction to the TSC). The owner will have to somehow return his NFT back.

Another drawback of the proposed solution is that it is not possible to quickly enumerate all the signatures issued by a given specific Provider authority. Each TSC depends not only on Provider's address and some integer index (like in a basic NFT collection contract) but also on the NFT object and signatory addresses.

# Rationale and alternatives
1. "One signature - one smart contract" simplifies interaction with signatures and allows to store any useful payload separately from the NFT object itself.
2. One NFT object can hold any number of signatures.
3. The signature is immutable, verifiable, and stored on the blockchain just like an NFT itself.

## Why not store signatures directly in the NFT contract?

To do this, one would have to develop a special separate extension of the existing NFT standard. Everyone who would like to use signatures for their NFTs should have minted it with a new smart contract. Moreover, each signature would have its own payload, so the NFT smart contract will contain more and more information with each new signature confirmed, and therefore consume more funds for storage.

## Why not consider a simple transaction to an NFT object as a signature?

Simple transactions, even if they are provided with special comments and payload, cannot serve as a sufficiently reasonable mechanism for signing NFT objects because of the following reasons:

1. Possible spamming of the NFT with microtransactions makes the detection of real conscious signatures rather difficult.
2. Possible smart contract phishing. A wallet owner can be tricked into sending a transaction to some NFT that is being “signed” without the knowledge of the signatory.
3. The complexity of indexing signatures in the blockchain for their subsequent validation.
4. The signature may contain additional information about the object and the signatory that are very difficult to standardize in case of a simple transaction, like a graphical representation, the ID of a linked social network account, etc.

# Prior art

# Unresolved questions

1. Should the possibility of a return of frozen NFTs be implemented?
Should we store the NFT owner's address in TSC in order to achieve this?

2. Should we treat TSC as a strict descendant of a standard NFT item with the standard interface?
