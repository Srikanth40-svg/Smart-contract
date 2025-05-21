# Smart-contract
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract TicketSale {
    address public admin;
    uint256 public ticketPrice;
    uint256 public maxTickets;
    uint256 public ticketsSold;
    bool public saleActive;
    
    mapping(address => uint256) public userTickets;
    
    event TicketPurchased(address indexed buyer, uint256 quantity, uint256 totalCost);
    event SaleStatusChanged(bool newStatus);
    event Withdrawal(address indexed admin, uint256 amount);
    
    modifier onlyAdmin() {
        require(msg.sender == admin, "Only admin can perform this action");
        _;
    }
    
    modifier saleIsActive() {
        require(saleActive, "Ticket sale is not active");
        _;
    }
    
    constructor(uint256 _ticketPrice, uint256 _maxTickets) {
        admin = msg.sender;
        ticketPrice = _ticketPrice;
        maxTickets = _maxTickets;
        saleActive = false; // Sale starts inactive by default
    }
    
    function buyTickets(uint256 quantity) external payable saleIsActive {
        require(quantity > 0, "Quantity must be greater than 0");
        require(ticketsSold + quantity <= maxTickets, "Not enough tickets available");
        require(msg.value == ticketPrice * quantity, "Incorrect ETH amount sent");
        
        userTickets[msg.sender] += quantity;
        ticketsSold += quantity;
        
        emit TicketPurchased(msg.sender, quantity, msg.value);
    }
    
    function setSaleStatus(bool _saleActive) external onlyAdmin {
        saleActive = _saleActive;
        emit SaleStatusChanged(_saleActive);
    }
    
    function withdraw() external onlyAdmin {
        uint256 balance = address(this).balance;
        require(balance > 0, "No funds to withdraw");
        
        (bool success, ) = admin.call{value: balance}("");
        require(success, "Withdrawal failed");
        
        emit Withdrawal(admin, balance);
    }
    
    function getAvailableTickets() external view returns (uint256) {
        return maxTickets - ticketsSold;
    }
    
    function getUserTicketBalance(address user) external view returns (uint256) {
        return userTickets[user];
    }
    
    // Additional feature: Change ticket price (admin only)
    function setTicketPrice(uint256 newPrice) external onlyAdmin {
        require(newPrice > 0, "Price must be greater than 0");
        ticketPrice = newPrice;
    }
    
    // Additional feature: Change max tickets (admin only)
    function setMaxTickets(uint256 newMax) external onlyAdmin {
        require(newMax >= ticketsSold, "Cannot set max below tickets already sold");
        maxTickets = newMax;
    }
    
    // Additional feature: Emergency refund (admin can refund a specific purchase)
    function issueRefund(address recipient, uint256 quantity) external onlyAdmin {
        require(userTickets[recipient] >= quantity, "Recipient doesn't have enough tickets");
        
        userTickets[recipient] -= quantity;
        ticketsSold -= quantity;
        
        uint256 refundAmount = quantity * ticketPrice;
        (bool success, ) = recipient.call{value: refundAmount}("");
        require(success, "Refund failed");
    }
}
