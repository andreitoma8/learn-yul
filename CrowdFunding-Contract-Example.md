# CrowdFunding Contracts in Yul and Solidity

The contract should be able to:

1. Allow anyone to create a crowdfunding campaign for themselves by setting:
    1. Minimum amount to be raised
    2. Duration
2. Donors can contribute to any crowdfunding campaign until the end timestamp of that campaign
3. If the minimum amount is raised at any time, the campaign owner can withdraw the money
4. If the duration is over and the money has not been raised, the users can withdraw their money

## Gas Consumption Comparison:

```
Gas  deploy   createCampaign  contribute  withdrawOwner  withdrawDonor
Yul  162282   110133          68026       37879          35132
Sol  503919   113384          68647       35685          67999
```

## Yul contract code:

```yul
/**
Storage Layout:
slot 0 = campaign ID counter
slot keccak256(campaign ID) = owner address
slot keccak256(campaign ID) + 1 = target amount
slot keccak256(campaign ID) + 2 = end timestamp
slot keccak256(campaign ID) + 3 = total raised
slot keccak256(donerAddress, campaign ID) - donations
*/

object "CrowdFunding" {
    code {
        // return the bytecode of the contract
        datacopy(0x00, dataoffset("runtime"), datasize("runtime"))
        return(0x00, datasize("runtime"))
    }

    object "runtime" {
        code {
            switch selector()
            case 0x22502268 /* createCampaign(uint256,uint256) target/duration */ {
                // require enough data is sent for params
                checkParamLenght(2)

                // get the campaign ID
                let id := sload(0x00)

                // store the ID to memory at 0x00
                mstore(0x00, id)

                // compute the storage slot of the campaing struct
                let structSlot := keccak256(0x00, 0x20)
                // store the campaign struct
                sstore(structSlot, caller())
                sstore(add(structSlot, 0x20), calldataload(4))
                sstore(add(structSlot, 0x40), add(timestamp() ,calldataload(36)))

                // increment the ID
                sstore(0x00, add(id, 1))
            }
            case 0xc1cbbca7 /* contribute(uint256) campaignId */ {
                // require enough data is sent for param
                checkParamLenght(1)
                // require value sent is more than 0
                require(lt(0, callvalue()))

                // get the id from calldata and store it in memory
                let id := calldataload(4)
                mstore(0x00, id)

                // require block timestamp lower than campaign end time
                let campaignStructSlot := keccak256(0x00, 0x20)
                let endTimestamp := sload(add(campaignStructSlot, 0x40))
                require(lt(timestamp(), endTimestamp))

                // sotre caller address to memory
                mstore(0x20, caller())

                // compute the storage slot of the invested value
                // of the user for this campaign
                let storageSlot := keccak256(0x00, 0x40)
                // get the already invested amount
                let alreadyInvested := sload(storageSlot)

                // store the new invested value
                sstore(storageSlot, add(alreadyInvested, callvalue()))

                // update total raised for campaign ID
                let totalStorageSlot := add(campaignStructSlot, 0x60)
                let previousTotal := sload(totalStorageSlot)
                sstore(totalStorageSlot, add(previousTotal, callvalue()))
            }
            case 0x6ef98b21 /* withdrawOwner(uint256) campaignId*/ {
                // require enough data is sent for params
                checkParamLenght(1)

                // store the campaing ID in memory
                mstore(0x00, calldataload(4))

                // get the storage slot of campaign struct
                let ownerStorageSlot := keccak256(0x00, 0x20)
                let targetAmountSlot := add(ownerStorageSlot, 0x20)
                let endTimestampSlot := add(ownerStorageSlot, 0x40)
                let amountRaisedSlot := add(ownerStorageSlot, 0x60)

                let amountRaised := sload(amountRaisedSlot)

                // require caller is the owner
                require(eq(caller(), sload(ownerStorageSlot)))
                // require minimum amount is raised
                require(lt(sload(targetAmountSlot), amountRaised))

                // delete the amount raised
                sstore(amountRaisedSlot, 0)

                // make end timestamp uint256.max so that
                // contributors can't withdraw
                sstore(endTimestampSlot, sub(0,1))

                // send value raised to owner
                if iszero(call(gas(), caller(), amountRaised, 0, 0, 0, 0)) {
                revert(0,0)
            }

            }
            case 0x152b58ab /* withdrawDonor(uint256) campaignId */{
                // require enough data is sent for params
                checkParamLenght(1)

                // store the campaing ID in memory
                mstore(0x00, calldataload(4))

                // get the storage slot of campaign struct
                let ownerStorageSlot := keccak256(0x00, 0x20)
                let targetAmountSlot := add(ownerStorageSlot, 0x20)
                let endTimestampSlot := add(ownerStorageSlot, 0x40)
                let amountRaisedSlot := add(ownerStorageSlot, 0x60)

                // require that the end timestamp has passed
                require(lt(sload(endTimestampSlot), timestamp()))
                // require that the target amount has not been raised
                require(lt(sload(amountRaisedSlot), sload(targetAmountSlot)))

                mstore(0x20, caller())

                // compute the storage slot of the invested value
                // of the user for this campaign
                let storageSlot := keccak256(0x00, 0x40)

                let amountToSend := sload(storageSlot)

                // require that the amount to send is bigger than 0
                require(lt(0, amountToSend))

                sstore(storageSlot, 0)

                // send back funds to doner
                transfer(amountToSend)
            }
            default {
                // if the function signature sent does not match any
                // of the contract functions, revert
                revert(0, 0)
            }

            // Return the function selector: the first 4 bytes of the call data
            function selector() -> s {
                s := div(calldataload(0), 0x100000000000000000000000000000000000000000000000000000000)
            }

            // Implementation of the require statement from Solidity
            function require(condition) {
                if iszero(condition) { revert(0, 0) }
            }

            // Check if the calldata has the correct number of params
            function checkParamLenght(len) {
                require(eq(calldatasize(), add(4, mul(32, len))))
            }

            // Transfer ether to the caller address
            function transfer(amount) {
                if iszero(call(gas(), caller(), amount, 0, 0, 0, 0)) {
                    revert(0,0)
                }
            }
        }
    }
}
```

## Solidity contract code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract CrowdFundingSolidity {
    uint256 id;

    struct Campaign {
        address owner;
        uint256 targetAmount;
        uint256 endTimestamp;
        uint256 amountRaised;
    }

    mapping(uint256 => Campaign) campaigns;
    mapping(address => uint256) donations;

    function createCampaign(uint256 target, uint256 duration) public {
        campaigns[id] = Campaign({
            owner: msg.sender,
            targetAmount: target,
            endTimestamp: block.timestamp + duration,
            amountRaised: 0
        });

        id++;
    }

    function contribute(uint256 campaignId) public payable {
        require(msg.value > 0);

        require(block.timestamp < campaigns[campaignId].endTimestamp);

        donations[msg.sender] += msg.value;

        campaigns[campaignId].amountRaised += msg.value;
    }

    function withdrawOwner(uint256 campaignId) public {
        Campaign memory campaign = campaigns[campaignId];
        require(msg.sender == campaign.owner);

        require(campaign.amountRaised > campaign. targetAmount);

        campaigns[campaignId].amountRaised = 0;

        (bool os, ) = payable(msg.sender).call{value: campaign.amountRaised}("");
        require(os);
    }

    function withdrawDonor(uint256 campaignId) public {
        require(donations[msg.sender] > 0);

        Campaign memory campaign = campaigns[campaignId];

        require(block.timestamp > campaign.endTimestamp);
        require(campaign.targetAmount < campaign.amountRaised);

        uint256 amountDonated = donations[msg.sender];
        donations[msg.sender] = 0;

        (bool os, ) = payable(msg.sender).call{value: amountDonated}("");
        require(os);
    }
}
```
