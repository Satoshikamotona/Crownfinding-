# Crownfinding-
This smart contract is an overview of a project I am working on for a client, it is based on crowdfunding and may contain some errors, I would like to get help

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {

        function transfer(address recipient, uint256 amount) external returns (bool);

    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
    function allowance(address owner, address spender) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
}

library SafeMath {
    function add(uint256 a, uint256 b) internal pure returns (uint256) {
        uint256 c = a + b;
        require(c >= a, "SafeMath: addition overflow");
        return c;
    }

    function sub(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b <= a, "SafeMath: subtraction overflow");
        uint256 c = a - b;
        return c;
    }

    function mul(uint256 a, uint256 b) internal pure returns (uint256) {
        if (a == 0) {
            return 0;
        }
        uint256 c = a * b;
        require(c / a == b, "SafeMath: multiplication overflow");
        return c;
    }

    function div(uint256 a, uint256 b) internal pure returns (uint256) {
        require(b > 0, "SafeMath: division by zero");
        uint256 c = a / b;
        return c;
    }
}

contract InfinixForceCrowdfunding {
    using SafeMath for uint256;

    address payable public owner;
    address public usdcAddress; // updated type to address payable
    uint256 public registrationFee = 1 * 10**18; // 1 USDC BEP20 in wei
    uint256 public totalContributions;
    uint256 public numberOfUsers;

    mapping(address => uint256) public userIDs;
    mapping(uint256 => address) public idToAddress;
    mapping(address => address) public referrers;
    mapping(address => uint256) public referralRewards;
    mapping(uint256 => bool) public paymentsMade;

    event Registration(address indexed user, address indexed directReferrer);

    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can call this function");
        _;
    }

    constructor(address payable _usdcAddress) {
        owner = payable (msg.sender);
        usdcAddress = _usdcAddress;
        numberOfUsers = 1;
        userIDs[owner] = numberOfUsers;
        idToAddress[numberOfUsers] = owner;
    }

    function register(address directReferrer) external {
        require(IERC20(usdcAddress).allowance(msg.sender, address(this)) >= registrationFee, "Insufficient allowance");
        require(IERC20(usdcAddress).transferFrom(msg.sender, address(this), registrationFee), "Transfer failed");
        require(userIDs[msg.sender] == 0, "User already registered");
        require(directReferrer != msg.sender, "Cannot be your own referrer");

        numberOfUsers = numberOfUsers.add(1);
        uint256 userID = numberOfUsers;
        userIDs[msg.sender] = userID;
        idToAddress[userID] = msg.sender;
        referrers[msg.sender] = directReferrer;

        // Distribute referral rewards
        _distributeReferralRewards(msg.sender, registrationFee);

        // Distribute random registration reward
        _distributeRandomRegistrationReward(registrationFee);

        // Update contribution tracking
        totalContributions = totalContributions.add(registrationFee);

        // Emit registration event
        emit Registration(msg.sender, directReferrer);
    }

    function _distributeReferralRewards(address user, uint256 amount) internal {
        address directReferrer = referrers[user];
        address indirectReferrer = referrers[directReferrer];
        address indirectReferrerOfReferrer = referrers[indirectReferrer];

        uint256 directReferrerReward = amount.mul(50).div(100);
        uint256 indirectReferrerReward = amount.mul(10).div(100);
        uint256 indirectReferrerOfReferrerReward = amount.mul(10).div(100);
        uint256 creatorReward = amount.mul(10).div(100);

        _transferWithReferralReward(indirectReferrer, indirectReferrerReward);
        _transferWithReferralReward(indirectReferrerOfReferrer, indirectReferrerOfReferrerReward);
        _transferWithReferralReward(directReferrer, directReferrerReward);

        // Distribute creator reward
        IERC20(usdcAddress).transfer(owner, creatorReward);
    }

    function _distributeRandomRegistrationReward(uint256 amount) internal {
        uint256 randomReward = amount.mul(20).div(100);
        uint256 randomUserID = uint256(keccak256(abi.encodePacked(block.timestamp, numberOfUsers))) % numberOfUsers + 1;
        address randomUser = idToAddress[randomUserID];
        _transferWithReferralReward(randomUser, randomReward);
        paymentsMade[randomUserID] = true;
    }

    function distributeEarnings() external onlyOwner {
        require(totalContributions > 0, "No contributions to distribute");
        require(IERC20(usdcAddress).balanceOf(address(this)) > 0, "Contract balance is zero");

        uint256 remainingReward = IERC20(usdcAddress).balanceOf(address(this)).mul(20).div(100);
        uint256 ownerReward = IERC20(usdcAddress).balanceOf(address(this)).mul(10).div(100);

        // Distribute remaining reward to 4 random users
        _distributeRandomRewards(remainingReward);

        // Distribute owner reward
        IERC20(usdcAddress).transfer(owner, ownerReward);

        totalContributions = 0; // Reset contributions after distribution
    }

    function _distributeRandomRewards(uint256 amount) internal {
        uint256 userLimit = numberOfUsers;
        if (userLimit > 4) {
            userLimit = 4;
        }

        for (uint256 i = 1; i <= userLimit; i++) {
            uint256 randomUserID = uint256(keccak256(abi.encodePacked(block.timestamp, i, numberOfUsers))) % numberOfUsers + 1;
            address randomUser = idToAddress[randomUserID];
            _transferWithReferralReward(randomUser, amount);
            paymentsMade[randomUserID] = true;
        }
    }

    function _transferWithReferralReward(address recipient, uint256 amount) internal {
        if (userIDs[recipient] != 0) {
            IERC20(usdcAddress).transfer(recipient, amount);
            referralRewards[recipient] = referralRewards[recipient].add(amount);
        } else {
            IERC20(usdcAddress).transfer(owner, amount);
        }
    }

 function changeOwner(address payable newOwner) external onlyOwner {
    require(newOwner != address(0), "Invalid address");
    owner = newOwner;
}

    receive() external payable {
        revert("Fallback function not allowed");
    }
}
