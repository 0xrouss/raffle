// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract Raffle is Ownable, ReentrancyGuard {
    IERC20 public token;
    uint256 public ticketPrice;
    uint256 public maxTickets;
    uint256 public totalTickets;
    uint256 public raffleEndTime;

    mapping(address => uint256) public ticketsOwned;
    address[] public participants;

    constructor(
        address _tokenAddress,
        uint256 _ticketPrice,
        uint256 _maxTickets,
        uint256 _raffleDuration
    ) Ownable(msg.sender) {
        token = IERC20(_tokenAddress);
        ticketPrice = _ticketPrice;
        maxTickets = _maxTickets;
        raffleEndTime = block.timestamp + _raffleDuration;
    }

    function buyTickets(uint256 _amount) public payable {
        require(block.timestamp < raffleEndTime, "Raffle has ended");
        require(msg.value == _amount * ticketPrice, "Incorrect Ether sent");
        require(totalTickets + _amount <= maxTickets, "Exceeds max tickets");

        if (ticketsOwned[msg.sender] == 0) {
            participants.push(msg.sender);
        }
        ticketsOwned[msg.sender] += _amount;
        totalTickets += _amount;
    }

    function pickWinner() external onlyOwner {
        require(block.timestamp >= raffleEndTime, "Raffle not ended yet");
        require(totalTickets > 0, "No tickets sold");

        // This is a simple and insecure way of generating randomness
        uint256 winnerIndex = uint256(keccak256(abi.encodePacked(block.timestamp, block.prevrandao))) % participants.length;
        address winner = participants[winnerIndex];

        uint256 prizeAmount = token.balanceOf(address(this));
        token.transfer(winner, prizeAmount);
    }

    function getTicketsOwned(address _user) public view returns (uint256) {
        return ticketsOwned[_user];
    }

    function getTimeLeft() public view returns (uint256) {
        if(block.timestamp >= raffleEndTime) {
            return 0;
        } else {
            return raffleEndTime - block.timestamp;
        }
    }

    function emergencyWithdraw() external onlyOwner {
        uint256 contractBalance = address(this).balance;
        payable(owner()).transfer(contractBalance);

        uint256 tokenBalance = token.balanceOf(address(this));
        token.transfer(owner(), tokenBalance);
    }
}
