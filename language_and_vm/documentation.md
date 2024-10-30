# Confidential Storage Extension for the Ethereum Virtual Machine (EVM)

### Introduction

Welcome to the documentation for our confidential storage extension to the Ethereum Virtual Machine (EVM). This extension introduces confidential storage capabilities, allowing developers to handle sensitive data securely within smart contracts. By building upon the existing EVM infrastructure, we've added minimal changes to ensure ease of adoption and maintain compatibility.

This documentation highlights the differences ("diff") from Cancun's Ethereum version to focus on the new features introduced. We recommend familiarizing yourself with the standard Ethereum documentation alongside this guide.

- - -

### Table of Contents

1.  [New Shielded Types](#1-new-shielded-types)
2.  [Storage Behavior](#2-storage-behavior)
3.  [Restrictions and Caveats](#3-restrictions-and-caveats)
    *   [3.1 No `public` Keyword for Shielded Variables](#31-no-public-keyword-for-shielded-variables)
    *   [3.2 No Shielded Constants](#32-no-shielded-constants)
    *   [3.3 Literals and Enums](#33-literals-and-enums)
    *   [3.4 Exponentiation and Gas Costs](#34-exponentiation-and-gas-costs)
    *   [3.5 `.min()` and `.max()` Functions](#35-min-and-max-functions)
    *   [3.6 `immutable` Variables](#36-immutable-variables)
    *   [3.7 Events](#37-events)
4.  [Casting and Type Conversion](#4-casting-and-type-conversion)
    *   [4.1 Explicit Casting Required](#41-explicit-casting-required)
    *   [4.2 Casting Addresses to `payable`](#42-casting-addresses-to-payable)
5.  [New Instructions](#5-new-instructions)
    *   [5.1 `CSTORE`](#51-cstore)
    *   [5.2 `CLOAD`](#52-cload)
6.  [Arrays and Collections](#6-arrays-and-collections)
    *   [6.1 Shielded Arrays](#61-shielded-arrays)
    *   [6.2 Limitations](#62-limitations)
    *   [6.3 Indexing Exception](#63-indexing-exception)
7.  [Best Practices](#7-best-practices)
8.  [Conclusion](#8-conclusion)
9.  [Feedback](#9-feedback)

- - -

### 1\. New Shielded Types

We introduce three new types called **shielded types**:

*   **`suint`**: Shielded unsigned integer.
*   **`sint`**: Shielded signed integer.
*   **`saddress`**: Shielded address.

These types function similarly to their unshielded counterparts (`uint`, `int`, `address`) but are designed to handle confidential data securely within the smart contract's storage.

#### Usage Example

```
contract ConfidentialWallet {
    suint256 confidentialBalance;
    saddress confidentialOwner;

    constructor(suint256 _initialBalance, saddress _owner) {
        confidentialBalance = _initialBalance;
        confidentialOwner = _owner;
    }

    function addFunds(suint256 amount) private {
        confidentialBalance += amount;
    }

    // Shielded public interface for balance inquiries
    function getConfidentialBalance(saddress caller) public view returns (suint256) {
        require(caller == confidentialOwner, "Unauthorized access");
        return confidentialBalance;
    }

    // Securely transfer funds from this wallet to another shielded address
    function confidentialTransfer(suint256 amount, saddress recipient) public {
        require(msg.sender == confidentialOwner, "Only the owner can transfer");
        require(confidentialBalance >= amount, "Insufficient balance");

        confidentialBalance -= amount;
        // `recipient` would use a shielded receive function to handle incoming funds
        // This line represents a private operation that modifies shielded storage
        // in another confidential contract instance.
        ConfidentialWallet(recipient).addFunds(amount);
    }
}
```


## 2\. Storage Behavior

*   **Whole Slot Consumption**: Shielded types consume an entire storage slot. This design choice ensures that a storage slot is entirely private or public, avoiding mixed storage types within a single slot.
    
*   **Future Improvements**: We plan to support slot packing for shielded types in future updates. Until then, developers can use inline assembly to achieve slot packing manually if necessary.
    

#### Assembly Slot Packing Example

```
contract RegularStorage {
    struct RegularStruct {
        uint64 a;   // Slot 0 (packed)
        uint128 b;  // Slot 0 (packed)
        uint64 c;   // Slot 0 (packed)
    }

    RegularStruct regularData;

    /* 
       Storage Layout:
       - Slot 0: [a | b | c]
    */
}

contract ShieldedStorage {
    struct ShieldedStruct {
        suint64 a;  // Slot 0
        suint128 b; // Slot 1
        suint64 c;  // Slot 2
    }

    ShieldedStruct shieldedData;

    /*
       Storage Layout:
       - Slot 0: [a]
       - Slot 1: [b]
       - Slot 2: [c]
    */
}
```

## 3\. Restrictions and Caveats

### 3.1 No `public` Keyword for Shielded Variables

*   Shielded variables **cannot** be declared as `public`. This restriction prevents accidental exposure of confidential data.
    
    `suint256 public confidentialNumber; // This will cause a compilation error`

### 3.2 No Shielded Constants

*   Shielded types **cannot** be used as constants. Constants are embedded in the contract bytecode, which is publicly accessible.
    
    `suint256 constant CONFIDENTIAL_CONSTANT = 42; // Not allowed`
    

### 3.3 Literals and Enums

*   Be cautious when using literals and enums with shielded types. They can inadvertently leak information if not handled properly.

### 3.4 Exponentiation and Gas Costs

*   Using shielded integers as exponents in exponentiation operations can leak information through gas usage, as gas cost scales with the exponent value.
    
### 3.5 `.min()` and `.max()` Functions

*   Calling `.min()` and `.max()` on shielded integers can reveal information about their values.

### 3.6 `immutable` Variables

*   Shielded `immutable` variables are only truly confidential if the transaction calldata used during their instantiation is encrypted.

### 3.7 Events

*   Shielded types **cannot** be emitted in events, as this would expose confidential data.
    
    `event ConfidentialEvent(suint256 confidentialData); // Not allowed`
    
*   **Future Improvements**: We plan to support encrypted events, enabling the emission of shielded types without compromising confidentiality.



## 4\. Casting and Type Conversion

### 4.1 Explicit Casting Required

*   Shielded types and their unshielded counterparts do **not** support implicit casting, except when using `suint` for array indexing.
    
```
uint256 publicNumber = 100;
suint256 confidentialNumber = suint256(publicNumber); // Explicit casting required`
``` 

### 4.2 Casting Addresses to `payable`

*   To cast an `saddress` to a `payable` address, use the following pattern:

```
address payable pay = payable(saddress(SomethingCastableToAnSaddress));`
```    


## 5\. New Instructions

We introduce two new EVM instructions to handle confidential storage:

### 5.1 `CSTORE`

*   **Purpose**: Stores shielded values in marked confidential storage slots.
*   **Behavior**: Sets the storage slot as confidential during the store operation.

### 5.2 `CLOAD`

*   **Purpose**: Retrieves shielded values from marked confidential or uninitialized storage slots.
*   **Behavior**: Only accesses storage slots marked as confidential.

### 5.3 Storage Rights Management

*   **Flagged Storage**: We introduce `FlaggedStorage` to tag storage slots as public or private based on the store instructions (`SSTORE` for public, `CSTORE` for confidential).
    
*   **Access Control**:
    
    *   **Public Storage**: Can be stored and loaded using `SSTORE` and `SLOAD`.
    *   **Confidential Storage**: Must be stored using `CSTORE` and loaded using `CLOAD`.
*   **Flexibility**: Storage slots are not permanently fixed as public or private. Developers can manage access rights using inline assembly if needed. Otherwise, the compiler will take care of it.
    

## 6\. Arrays and Collections

### 6.1 Shielded Arrays

*   Arrays of shielded types are supported, and their metadata (e.g., length of a dynamic array) is also stored in confidential storage.
    
`suint256[] private confidentialDynamicArray;`

### 6.2 Limitations

*   Currently, shielded arrays only work with the shielded types (`suint`, `sint`, `saddress`).
*   Shielded `bytes` or `string` arrays are **not yet supported**.

### 6.3 Shielded Indexing

*   Shielded unsigned integers (`suint`) can be used for array indexing without explicit casting.
 


## 7\. Best Practices

*   **Avoid Public Exposure**: Never expose shielded variables through public getters or events.
*   **Careful with Gas Usage**: Be mindful of operations where gas cost can vary based on shielded values (e.g., loops, exponentiation).
*   **Encrypt Calldata**: Ensure that any calldata used for initializing shielded `immutable` variables is encrypted.
*   **Manual Slot Packing**: If slot packing is necessary, use inline assembly carefully to avoid introducing vulnerabilities.
*   **Review Compiler Warnings**: Pay attention to compiler warnings related to shielded types to prevent accidental leaks.


## 8\. Conclusion

This extension enhances the EVM by introducing confidential storage capabilities, allowing developers to handle sensitive data securely. By understanding the new shielded types, instructions, and associated caveats, you can leverage these features to build more secure smart contracts.

We encourage you to refer to the standard Ethereum documentation for foundational concepts and use this guide to understand the differences and new functionalities introduced.


## 9\. Feedback

We welcome your feedback on this documentation. If you have suggestions or encounter any issues, please contact our support team or contribute to our documentation repository.

- - -
