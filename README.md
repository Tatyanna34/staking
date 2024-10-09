# staking// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

interface IERC20 {
    function transfer(address recipient, uint256 amount) external returns (bool);
    function transferFrom(address sender, address recipient, uint256 amount) external returns (bool);
    function balanceOf(address account) external view returns (uint256);
}

contract SimpleStaking {
    IERC20 public stakingToken;        // ERC20 token used for staking
    uint256 public rewardRate;         // Reward rate per second (e.g., 1 token per second)
    uint256 public totalStaked;        // Total amount of tokens staked in the contract
    uint256 public lastUpdateTime;     // The last time rewards were updated

    mapping(address => uint256) public stakedBalance;  // Amount of tokens staked by each user
    mapping(address => uint256) public rewardBalance;  // Rewards earned by each user
    mapping(address => uint256) public userRewardPerTokenPaid;  // Track user's rewards per token paid

    constructor(IERC20 _stakingToken, uint256 _rewardRate) {
        stakingToken = _stakingToken;
        rewardRate = _rewardRate;
        lastUpdateTime = block.timestamp;
    }

    // Update reward for the user
    modifier updateReward(address _account) {
        rewardBalance[_account] = earned(_account);
        userRewardPerTokenPaid[_account] = rewardPerToken();
        lastUpdateTime = block.timestamp;
        _;
    }

    // Calculate reward per token
    function rewardPerToken() public view returns (uint256) {
        if (totalStaked == 0) {
            return 0;
        }
        return (block.timestamp - lastUpdateTime) * rewardRate / totalStaked;
    }

    // Calculate the rewards earned by a user
    function earned(address _account) public view returns (uint256) {
        return stakedBalance[_account] * (rewardPerToken() - userRewardPerTokenPaid[_account]) / 1e18
            + rewardBalance[_account];
    }

    // Stake tokens
    function stake(uint256 _amount) external updateReward(msg.sender) {
        require(_amount > 0, "Cannot stake 0");
        stakingToken.transferFrom(msg.sender, address(this), _amount);
        totalStaked += _amount;
        stakedBalance[msg.sender] += _amount;
    }

    // Withdraw staked tokens and rewards
    function withdraw(uint256 _amount) public updateReward(msg.sender) {
        require(_amount > 0, "Cannot withdraw 0");
        require(stakedBalance[msg.sender] >= _amount, "Insufficient balance");
        totalStaked -= _amount;
        stakedBalance[msg.sender] -= _amount;
        stakingToken.transfer(msg.sender, _amount);
    }

    // Claim earned rewards
    function claimReward() public updateReward(msg.sender) {
        uint256 reward = rewardBalance[msg.sender];
        require(reward > 0, "No reward to claim");
        rewardBalance[msg.sender] = 0;
        stakingToken.transfer(msg.sender, reward);
    }

    // Exit staking (withdraw staked tokens and claim rewards)
    function exit() external {
        withdraw(stakedBalance[msg.sender]);
        claimReward();
    }
}
