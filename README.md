# SlotMachine
BaseSlotMachine.sol
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.34;

contract BaseSlotMachine {
    address public immutable owner;
    uint256 public treasuryBalance;

    event SpinResult(
        address indexed player,
        uint8[3] symbols,
        uint256 bet,
        uint256 payout,
        uint256 timestamp
    );

    // 0=🍒 1=🔔 2=⭐ 3=7 4=💎
    constructor() {
        owner = msg.sender;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only owner can call this");
        _;
    }

    function spin() external payable {
        require(msg.value == 0.001 ether, "Must pay exactly 0.001 ETH to spin");

        uint8[3] memory symbols;
        for (uint i = 0; i < 3; i++) {
            symbols[i] = uint8(uint256(keccak256(abi.encodePacked(
                block.prevrandao,
                msg.sender,
                block.timestamp,
                i
            ))) % 5);
        }

        uint256 payout = 0;

        // rule
        if (symbols[0] == symbols[1] && symbols[1] == symbols[2]) {
            if (symbols[0] == 4) payout = 0.01 ether;      // 3💎 = 10x
            else if (symbols[0] == 3) payout = 0.005 ether; // 3 7 = 5x
            else payout = 0.003 ether;                       // the same 3 = 3x
        } else if (symbols[0] == symbols[1] || symbols[1] == symbols[2] || symbols[0] == symbols[2]) {
            payout = 0.0015 ether;                           // 2same = 1.5x
        } else {
            treasuryBalance += msg.value;                    // false
        }

        if (payout > 0) {
            payable(msg.sender).transfer(payout);
        }

        emit SpinResult(msg.sender, symbols, msg.value, payout, block.timestamp);
    }

    function getTreasury() external view returns (uint256) {
        return treasuryBalance;
    }

    function withdrawTreasury() external onlyOwner {
        uint256 amount = treasuryBalance;
        treasuryBalance = 0;
        payable(owner).transfer(amount);
    }

    function getContractInfo() external pure returns (string memory) {
        return "BaseSlotMachine - Pay 0.001 ETH to spin and win up to 10x!";
    }
}
