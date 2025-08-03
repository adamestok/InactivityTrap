# FineBalanceShiftTrap - is a custom trap for a Drosera node designed to monitor balance anomalies at a given address.


**FineBalanceShiftTrap — Drosera Trap SERGEANT and CAPTAIN** 

# Objective

The trap monitors minor changes in the balance of the specified wallet and is triggered if the difference between the current and previous values exceeds the specified sensitivity threshold (default is 0.5%).
# Problem

monitoredWallet — the target address being monitored.

sensitivity — the sensitivity threshold in ppm (‰). A value of 5 means 0.5%.

The collect() function returns the current balance of the monitored address.

The shouldRespond() function analyzes the last two balances and determines whether there is a deviation greater than the specified threshold.
# Trap Logic Summary

The last two balance values are compared.

The percentage of change in ppm is calculated.

If the change is equal to or exceeds the threshold, the trap is activated.


# contract FineBalanceShiftTrap
}
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ITrap {
    function collect() external view returns (bytes memory);
    function shouldRespond(bytes[] calldata data) external pure returns (bool, bytes memory);
}

contract FineBalanceShiftTrap is ITrap {
    address public constant monitoredWallet = 0x53eD4A8B4D93e9Ab7ee211a34F9C439024c5Ec8c;
    uint256 public constant sensitivity = 5;

    function collect() external view override returns (bytes memory) {
        return abi.encode(monitoredWallet.balance);
    }

    function shouldRespond(bytes[] calldata data) external pure override returns (bool, bytes memory) {
        if (data.length < 2) return (false, "Insufficient data");

        uint256 latest = abi.decode(data[0], (uint256));
        uint256 previous = abi.decode(data[1], (uint256));

        if (previous == 0) return (false, "Previous balance is zero");

        uint256 diff = latest > previous ? latest - previous : previous - latest;
        uint256 permilleChange = (diff * 1000) / previous;

        if (permilleChange >= sensitivity) {
            return (true, "");
        }

        return (false, "");
    }
}


# Response Contract: BalanceShiftReceiver.sol
```
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract BalanceShiftReceiver {
    event BalanceShiftDetected(string details);

    function notifyShift(string calldata details) external {
        emit BalanceShiftDetected(details);
    }
}
}}


## What It Solves 

The FineBalanceShiftTrap event logs:

-Who was monitored (monitoredAddress),

-Time (block.timestamp),

-Alarm message.

The logInactivity() function will be called by Drosera nodes when the trap is triggered.
---

# Deployment & Setup Instructions 

1. ## _Deploy Contracts (e.g., via Foundry)_ 
```
forge create src/FineBalanceShiftTrap.sol:FineBalanceShiftTrap \
  --rpc-url https://ethereum-hoodi-rpc.publicnode.com \
  --private-key 0x...
```
```
2. ## _Update drosera.toml_ 
```
[traps.mytrap]
path = "out/FineBalanceShiftTrap.sol/FineBalanceShiftTrap.json"
response_contract = "<BalanceShiftReceiver address>"
response_function = "logAnomaly(string)"
```
3. ## _Apply changes_ 
```
DROSERA_PRIVATE_KEY=0x... drosera apply
```
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

