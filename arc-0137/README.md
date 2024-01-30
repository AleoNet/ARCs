---
arc: 137
title: Aleo Domain Name Service (ANS) - Specification
authors: snowtigersoft
discussion: https://github.com/AleoHQ/ARCs/discussions/45
topic: Application
status: Draft
created: 2023-11-12
---

## Abstract

This ARC proposes the Aleo Name Service (ANS), a protocol and program definition designed to resolve short, human-readable names to service and resource identifiers on the Aleo network. 

ANS aims to simplify the interaction with Aleo's resources by allowing memorable and updatable human-readable identifiers. 

It supports public and private domain names, each serving distinct use cases and privacy needs.

The goal of public domain names is to provide stable, human-readable identifiers that can be used to specify network resources. In this way, users can enter a memorable string, such as ‘snowtiger.ans’ or ‘wallet.leo’, and be directed to the appropriate resource.

For a private domain name, users can transfer Aleo Credits(ACs) to a ANS domain, and the holder can claim the ACs without let others know the real aleo address.

ANS 

## Specification

### Overview

The ANS protocol comprises three primary components:

- **ANS Registry Program**: Manages the domain name system, mapping names to resolvers and allowing updates to these mappings.
- **Registrars**: Assign domain names within the ANS, ensuring efficient and secure name distribution.
- **Resolvers**: Retrieve resource information associated with a name, such as contract addresses, content hashes, or IP addresses.

### ANS Registry

- **Functionality**: Enables domain owners to set resolver addresses and create subdomains.
- **Administration**: Includes a `register_tld` method to assign TLDs to registrars.

### ANS Resolvers

- **Purpose**: Handle resource lookups for names, returning the requested data.
- **Flexibility**: Different resolver types can be implemented for varied resources.

### ANS Registrars

- **Domain Allocation**: Distribute domain names to users.
- **TLD Management**: Specific TLDs can be managed by designated registrars.

### Name Syntax

ANS names adhere to the following syntax:

```
<domain> ::= <label> | <domain> "." <label>
<label> ::= [0-9a-z-_]{1,64}
```

### Namehash Algorithm

The `namehash` algorithm is implemented in the Leo programming language as follows:

```leo
struct Name {
    name: [u128; 4],
    parent: field
}

// The namehash is calc with : Poseidon2::hash_to_field(Name)
let name_hash: field = Poseidon2::hash_to_field(Name);

// Example hashes:
// snowtiger.ans -> 4039165989542226292217059306495671259264448213602066982803038338315267964460field
// aleo.ans -> 7754978972835365012111155851647542017963585681842055864179947501328380929370field
```

### Registry Specification

The ANS Registry is the central component of the Aleo Name Service (ANS) that maps human-readable names to resource identifiers. It allows for the registration, transfer, and management of domain names.

#### Structs

```leo
// Struct representing a domain name within the ANS.
// The `name` field holds the ASCII bits of a domain name. If the length of the bits is less than 512,
// zeros are appended to the end. The bits are then split into four parts, with `name[0]` holding the
// first 128 bits and `name[3]` holding the last 128 bits.
struct Name {
    name: [u128; 4],
    parent: field // The name_hash of the parent name.
}

// Extends the `Name` struct to include the resolver address.
// The `resolver` field points to the resolver program responsible for handling queries related to the domain.
// A value of 0u128 for the resolver indicates that the current program is used as the resolver.
struct NameStruct {
    name: [u128; 4],
    parent: field,
    resolver: u128 // The resolver program address.
}
```

#### Mappings
```leo
// Mapping from a name_hash to the owner address.
// Used to track the ownership of NFTs representing public domain names.
mapping nft_owners: field => address;

// Mapping from a name_hash to its corresponding NameStruct.
mapping names: field => NameStruct;
```

#### Functions
Below are detailed descriptions for each function within the registry specification:
```leo
// Function to register a top-level domain (TLD) with the ANS Registry.
// Only administrators can invoke this function.
// @param registrer: The address of the registrar assigned to the TLD.
// @param tld: The TLD to be registered.
transition register_tld(registrer: address, tld: [u128; 4])

// Allows the registration of a domain name by verifying the ownership of the parent ANS name.
// The caller must be the owner of the parent domain(an authorized program or address).
// @param name: The domain name to be registered.
// @param parent: The name_hash of the parent domain.
// @param receiver: The address to receive the NFT representing the domain ownership.
// @param resolver: The resolver program address.
// @return NFT: The NFT record representing the new domain registration.
transition register(name: [u128; 4], parent: field, receiver: address, resolver: u128) -> NFT

// Registers a subdomain where the parent domain is a private domain name represented by an NFT record.
// @param name: The domain name to be registered.
// @param parent: The NFT representing the parent domain.
// @param receiver: The address to receive the NFT representing the subdomain ownership.
// @param resolver: The resolver program address.
// @return (NFT, NFT): The NFT records representing the parent and new domain registration.
transition register_private(name: [u128; 4], parent: NFT, receiver: address, resolver: u128) -> (NFT, NFT)

// the difference between register_public and register is that register_public does not require the caller to be the owner of the parent name
// instead, the signer must be the owner of the parent name
// that means, the owner of the parent name must be an account not a program
// @param name: The domain name to be registered.
// @param parent: The name_hash of the parent domain.
// @param receiver: The address to receive the NFT representing the domain ownership.
// @param resolver: The resolver program address.
// @return NFT: The NFT record representing the new domain registration.
transition register_public(name: [u128; 4], parent: field, receiver: address, resolver: u128) -> NFT

// Function to transfer the ownership of a private domain name represented by an NFT.
// @param nft: The NFT record representing the domain to be transferred.
// @param receiver: The address to receive the ownership of the domain.
// @return NFT: The updated NFT record with the new owner.
transition transfer_private(nft: NFT, receiver: address) -> NFT

// Function to transfer the ownership of a public domain name.
// @param receiver: The address to receive the ownership of the domain.
// @param name_hash: The hashed representation of the name.
transition transfer_public(receiver: address, name_hash: field)

// Function to convert a private domain name represented by an NFT to public domain name, 
// and transfer the ownership of the public domain name to the receiver, the receiver can be the original owner of the private domain name
// @param nft: The NFT record representing the domain to be converted.
// @param receiver: The address to receive the ownership of the domain.
transition convert_private_to_public(nft: NFT, receiver: address)

// Function to convert a public domain name to private domain name represented by an NFT,
// and transfer the ownership of the private domain name to the receiver, the receiver can be the original owner of the public domain name.
// @param name_hash: The hashed representation of the name.
// @param receiver: The address to receive the ownership of the domain.
// @return NFT: The NFT record with the new owner.
transition convert_public_to_private(name_hash: field, receiver: address) -> NFT

// Function to check the ownership of a name.
// This function is used by other programs to verify if an address owns a particular public name.
// @param name_hash: The hashed representation of the name.
// @param owner: The address suspected of owning the name.
transition check_ownership(name_hash: field, owner: address)

```

### Registrar Specification

Registrar only needs to implement a register_fld function. 

```leo
// register a first level domain
// the self.caller must be the holder of parent ANS name, it can be a program or an account
transition register_fld(
    name: [u128; 4],
    receiver: address,
    pay_record: credits.leo/credits
) {
    // this is "ans" name hash, different TLDs have different name hash, and assigned to different registrars
    let parent: field = 3601410589032411677092457044111621862970800028849492457114786804129430260029field;
    aleo_name_service_registry_v2.leo/register(name, parent, receiver, 0u128);
    return then finalize();
}

```

### Resolver Specification

The Resolver is a critical component of the Aleo Name Service (ANS) that translates human-readable names into machine-readable identifiers such as Aleo addresses, other cryptocurrency addresses, IPFS hash pointers for data, and more.

#### Structs

```leo
// Struct to index resolvers by the name_hash and a category for the type of resource they resolve to.
// This allows for flexibility in the type of data associated with a name.
struct ResolverIndex {
    name: field, // The name_hash of the domain.
    category: u128, // The type of the resolver, indicating the resource category.
    version: u64 // The version of the resolver, useful for updates and cache invalidation.
}
```

#### Mappings

```leo
// Mapping from an Aleo address to a primary name_hash. This is used to look up the primary name associated with an address.
mapping primary_names: address => field;

// Mapping from a ResolverIndex to the resolver data. The data typically includes the resource record associated with a name.
mapping resolvers: ResolverIndex => [u128; 4];

// Mapping from a name_hash to its current version number. This is used to manage resolver versioning.
mapping name_versions: field => u64;
```

#### Functions

Below are detailed descriptions for each function within the resolver specification:

```leo
// Sets the primary name associated with the caller's address.
// @param name_hash: The name_hash of the domain to be set as primary.
transition set_primary_name(name_hash: field)

// Unsets the primary name associated with the caller's address.
transition unset_primary_name()

// Updates the resolver address for a given name_hash in the names mapping.
// @param name_hash: The name_hash of the domain whose resolver is being set.
// @param resolver: The new resolver program address.
transition set_resolver(name_hash: field, resolver: u128)

// Adds or updates a resolver record for a given name_hash with a specified category.
// @param name_hash: The name_hash of the domain to update the resolver record for.
// @param category: A u128 bit representation of the category type for the record.
// @param content: The data associated with the resolver record, such as an address or hash.
transition set_resolver_record(name_hash: field, category: u128, content: [u128; 4])

// Removes a resolver record for a given name_hash and category.
// @param name_hash: The name_hash of the domain to remove the resolver record from.
// @param category: A u128 bit representation of the category type for the record being removed.
transition unset_resolver_record(name_hash: field, category: u128)

// Clears all resolver records associated with a given name_hash.
// This action is often used to reset the state of a domain's resolution data.
// Clearing of resolver records by incrementing the version number.
// This ensures that all caches will consider the resolver records for this name_hash as expired.
// @param name_hash: The name_hash of the domain to clear resolver records for.
transition clear_resolver_record(name_hash: field)
```

## Privacy Credit Transfer Scheme

The Privacy Credit Transfer Scheme is an innovative feature built upon the Aleo Name Service (ANS) that facilitates the private transfer of credits. This scheme ensures that neither party in the transaction is required to disclose their Aleo address, thereby enhancing privacy while maintaining ease of use.

### Mappings

```leo
// Mapping from a name_hash to the secret associated with the domain.
// This secret is used to verify the claim of the credits transferred to the domain.
mapping domain_credits: field => u64;
```

### Transfer Credits

This function enables the transfer of credits to an ANS domain without revealing the recipient's real Aleo address. It involves a secret that allows the domain holder to claim the transferred credits privately.

```leo
// Function for transferring credits to an ANS domain without revealing the domain holder's real address.
// @param receiver: The name_hash of the recipient ANS domain.
// @param secret: The secret associated with the transaction, used for claim verification.
// @param amount: The amount of credits to be transferred.
// @param pay_record: The record of the payment being made.
transition transfer_credits(receiver: field, secret: [u128; 4], amount: u64, pay_record: credits.leo/credits)
```

### Claim Credits

These functions allow domain holders to claim the transferred credits. Depending on whether the domain is public or private, the appropriate claim function is used.

Claim Credits Private
```leo
// Function for a domain holder to claim credits using a private ANS domain represented by an NFT and a secret.
// This ensures that the claim process remains private and the domain holder's real address is not exposed.
// @param nft: The NFT record representing the private ANS domain.
// @param secret: The secret used to verify the claim.
// @param amount: The amount of credits to be claimed.
transition claim_credits_private(nft: NFT, secret: [u128; 4], amount: u64)
```

Claim Credits Public
```leo
// Similar to the private claim function, this enables the claiming of credits for a public ANS domain.
// The domain holder uses a secret to claim the credits, maintaining privacy.
// @param name_hash: The name_hash of the public ANS domain.
// @param secret: The secret used to verify the claim.
// @param amount: The amount of credits to be claimed.
transition claim_credits_public(name_hash: field, secret: [u128; 4], amount: u64)
```

## Compatibility with ARC-0721

The Aleo Name Service (ANS) aims to be as compatible as possible with the ARC-0721 standard ([AleoHQ/ARCs/discussions/36](https://github.com/AleoHQ/ARCs/discussions/36)), which establishes a framework for non-fungible tokens on the Aleo platform. However, there are some irreconcilable differences between the two, primarily due to the unique requirements of the ANS.

### Divergence in NFT Structure

One of the key differences lies in the structure of the NFT used within ANS. While ARC-0721 defines a standard structure for NFTs, ANS requires a dynamic approach to the `data` field within the NFT record. This is due to the nature of domain name registration, where each NFT must reflect a unique identifier (name_hash) that is only determined at the time of domain registration by the user. Below is the ANS-specific NFT structure:

```leo
// The ANS NFT structure diverges from ARC-0721 in the `data` field.
// Here, `data` is not predefined but is dynamically created based on the domain name registered by the user.
// This `data` serves as the name_hash of the name, uniquely identifying the domain within ANS.
record NFT {
    owner: address,
    data: field,  // The dynamic name_hash of the registered domain name.
    edition: scalar // The edition number, similar to ARC-0721's structure.
}
```

### Embracing Differences for Enhanced Functionality

The modifications to the NFT structure within ANS are necessary to support the protocol's functionality and objectives. While ANS strives to align with existing standards like ARC-0721, it also recognizes the need to innovate and adapt its NFT representation to serve its unique purpose effectively. This approach ensures that ANS can provide a robust and privacy-centric naming service that complements the broader Aleo ecosystem.

It is important for the community and developers to be aware of these differences for a seamless integration and to leverage the strengths of both standards where they apply.

## Reference Implementations

1. Regitry: https://github.com/S-T-Soft/ans-programs/blob/main/programs/registry/src/main.leo
2. Registrar: https://github.com/S-T-Soft/ans-programs/blob/main/programs/ansregistrar/src/main.leo
3. Credit Transfer: https://github.com/S-T-Soft/ans-programs/blob/main/programs/credit_transfer/src/main.leo

## Dependencies

As this is an application ARC, there are no dependencies other than what is currently available in Aleo.

### Backwards Compatibility

Not necessary.


## Security & Compliance

This is an application ARC standard, so this only affects the security of managing assets on-chain.


## References

1. [ERC-137: Ethereum Domain Name Service - Specification](https://eips.ethereum.org/EIPS/eip-137)
2. [ARC-0721: Aleo Non-Fungible Token Standard](https://github.com/AleoHQ/ARCs/discussions/36)