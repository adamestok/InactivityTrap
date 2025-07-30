# InactivityTrap
**Inactivity Trap— Drosera Trap SERGEANT** 

# Objective

-Monitoring a validator contract that should always be active.

-Checking timeouts in a DeFi strategy.

-Can be used as a watchdog "trap for a watchdog".
# Problem

-collect() returns block.timestamp (i.e. the current time of the call).

-shouldRespond() compares two timestamps (current and previous).

-If more than 1 hour has passed without activity, shouldRespond() returns true.
---

# Trap Logic Summary

_Trap Contract: InactivityTrap.sol_

Monitoring that there is at least one transaction from a controlled address in a given period of time (e.g. 1 hour). If activity disappears, something suspicious may have happened (e.g. an attacker stopped an automated script).
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract InactivityTrap {
    address public constant target = 0x53eD4A8B4D93e9Ab7ee211a34F9C439024c5Ec8c;
    uint256 public constant inactivityThreshold = 3600; // 1 hour

    struct CollectOutput {
        uint256 timestamp;
        uint256 targetNonce;
    }

    constructor() {}

    // Collect current timestamp and target's nonce (proxy for activity)
    function collect() external view returns (bytes memory) {
        uint256 nonce = target.code.length == 0
            ? targetNonceEOA()
            : block.number; // placeholder for contract activity (could check events)
        return abi.encode(CollectOutput(block.timestamp, nonce));
    }

    // Compare with previous sample
    function shouldRespond(bytes[] calldata data) external pure returns (bool, bytes memory) {
        if (data.length < 2) return (false, bytes(""));

        CollectOutput memory current = abi.decode(data[0], (CollectOutput));
        CollectOutput memory previous = abi.decode(data[1], (CollectOutput));

        // No change in nonce for too long
        if (current.timestamp - previous.timestamp >= inactivityThreshold
            && current.targetNonce == previous.targetNonce) {
            return (true, bytes(""));
        }

        return (false, bytes(""));
    }

    function targetNonceEOA() internal view returns (uint256) {
        return target.balance; // placeholder, consider getting real tx nonce via RPC or logs
    }
}



# Response Contract: LogAlertReceiver.sol
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LogAlertReceiver {
    event InactivityDetected(address monitoredAddress, uint256 timestamp);

    function logInactivity(address monitoredAddress) external {
        emit InactivityDetected(monitoredAddress, block.timestamp);
    }
}


# What It Solves 

The InactivityDetected event logs:

-Who was monitored (monitoredAddress),

-Time (block.timestamp),

-Alarm message.

The logInactivity() function will be called by Drosera nodes when the trap is triggered.
---

# Deployment & Setup Instructions 

1. ## _Deploy Contracts (e.g., via Foundry)_ 
```
forge create src/InactivityTrap.sol: InactivityTrap \
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \
  --private-key 0x...
```
```
forge create src/LogAlertReceiver.sol:LogAlertReceiver \
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \
  --private-key 0x...
```
2. ## _Update drosera.toml_ 
```
[traps.mytrap]
path = "out/InactivityTrap.sol/InactivityTrap.json"
response_contract = "<LogAlertReceiver address>"
response_function = "logAnomaly(string)"
```
3. ## _Apply changes_ 
```
DROSERA_PRIVATE_KEY=0x... drosera apply
```

<img width="547" height="354" alt="{F13A1A68-C0D0-4DFE-AF0B-03BA506B3899}" src="https://github.com/user-attachments/assets/e4ad73d9-5d32-4b65-9803-41c9872d3e4c" />


# Testing the Trap 

1. Send ETH to/from target address on Ethereum Hoodi testnet.

2. Wait 1-3 blocks.

3. Observe logs from Drosera operator:

4. get ShouldRespond='true' in logs and Drosera dashboard
---

# Extensions & Improvements 

- Allow dynamic threshold setting via setter,

- Track ERC-20 balances in addition to native ETH,

- Chain multiple traps using a unified collector.


# Date & Author

_First created: July 26, 2025_

## Author: Adamestok && Profit_Nodes 
TG : @mestuq

Discord: naberyn

