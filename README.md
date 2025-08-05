🛡️ SafeFlowTrap — Ethereum Balance Anomaly Trap

Overview

SafeFlowTrap is a lightweight, on-chain anomaly detection contract for Drosera that monitors the native ETH balance of a specific wallet address.
If the balance drops sharply or spikes abnormally, the trap triggers and sends an alert via a connected LogAlertReceiver contract.

This contract is designed for DAO treasury monitoring, high-value wallets, or any Ethereum address where sudden balance anomalies may indicate unexpected activity.

Features

* Fixed target address – trap is locked to a single wallet
* Drop detection: triggers if balance decreases by 30% or more compared to the previous observation.
* Spike detection: triggers if balance increases by 200% or more compared to the previous observation.
* Minimal deployment: no constructor parameters (compatible with Drosera’s apply process).
* Log emission: alerts are sent to a LogAlertReceiver contract and can be collected from chain logs.


Architecture

+----------------+        Monitor        +---------------------+
|  SafeFlowTrap  | --------------------> |  LogAlertReceiver   |
+----------------+                       +---------------------+
        |                                           |
   (Collect balance)                          (Emit `Alert` event)

* ITrap interface: exposes collect() and shouldRespond() for the Drosera operator.
* ILogAlertReceiver: simple interface for logging anomaly messages.

Contracts
SafeFlowTrap.sol



// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface ITrap {
    function collect() external view returns (bytes memory);
    function shouldRespond(bytes[] calldata data) external pure returns (bool, bytes memory);
}

interface ILogAlertReceiver {    
    function logAnomaly(string calldata message) external;
}

contract SafeFlowTrap is ITrap {
    // Fixed monitored address
    address public constant target = 0xABcDEF1234567890abCDef1234567890AbcDeF12;

    // Configurable receiver for alert logging
    address public alertReceiver;

    uint256 public constant DROP_THRESHOLD_PERCENT = 30;
    uint256 public constant SPIKE_THRESHOLD_PERCENT = 200;

    /// @notice sets alert receiver (only once)
    function setAlertReceiver(address _receiver) external {
        require(alertReceiver == address(0), "Receiver already set");
        require(_receiver != address(0), "Invalid receiver");
        alertReceiver = _receiver;
    }

    function collect() external view override returns (bytes memory) {
        return abi.encode(target.balance);
    }

    function shouldRespond(bytes[] calldata data) external pure override returns (bool, bytes memory) {
        if (data.length < 2) return (false, "Insufficient data");
        uint256 current = abi.decode(data[0], (uint256));
        uint256 previous = abi.decode(data[1], (uint256));
        if (previous == 0) return (false, "No previous data");
        uint256 diff = current > previous ? current - previous : previous - current;
        uint256 percent = (diff * 100) / previous;

        if (current < previous && percent >= DROP_THRESHOLD_PERCENT)
            return (true, abi.encode("Balance DROP anomaly detected"));

        if (current > previous && percent >= SPIKE_THRESHOLD_PERCENT)
            return (true, abi.encode("Balance SPIKE anomaly detected"));

        return (false, "");
    }

    /// @dev Called by Drosera operator upon confirmed anomaly
    function alert(string calldata message) external {
        require(alertReceiver != address(0), "Receiver not set");
        ILogAlertReceiver(alertReceiver).logAnomaly(message);
    }
}


LogAlertReceiver.sol

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract LogAlertReceiver {
    event Alert(string message);

    function logAnomaly(string calldata message) external {
        emit Alert(message);
    }
}


Deployment

1. Deploy both contracts:

cast send --private-key $PK --rpc-url $RPC <Bytecode>

2. Set the alert receiver:

drosera trap-call <trap_address> setAlertReceiver <log_receiver_address>

3. Apply the trap to the network:

DROSERA_PRIVATE_KEY=$PK drosera apply


Logs Example
When a significant balance change is detected:

INFO drosera_services::operator::enzyme::runner: ShouldRespond='true'
INFO drosera_services::operator::submission::runner: submitting to network

And the receiver contract emits:

event Alert("Balance DROP anomaly detected");


