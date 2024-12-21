// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract PredictionMarket {

    address public owner;
    uint256 public predictionCount;

    struct Prediction {
        string question;
        uint256 closingTime;
        uint256 outcome;  // 0 for undecided, 1 for correct, 2 for incorrect
        uint256 totalAmount;
        mapping(address => uint256) bets;
    }

    // Mapping for predictions by ID
    mapping(uint256 => Prediction) public predictions;

    event PredictionCreated(uint256 predictionId, string question, uint256 closingTime);
    event BetPlaced(uint256 predictionId, address bettor, uint256 amount);
    event PredictionResolved(uint256 predictionId, uint256 outcome);
    event RewardClaimed(uint256 predictionId, address claimant, uint256 amount);

    constructor() {
        owner = msg.sender;
        predictionCount = 0;
    }

    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can perform this action");
        _;
    }

    // Function to create a new prediction market
    function createPrediction(string memory _question, uint256 _closingTime) public onlyOwner {
        predictions[predictionCount].question = _question;
        predictions[predictionCount].closingTime = _closingTime;
        predictions[predictionCount].outcome = 0; // Outcome undecided initially
        predictions[predictionCount].totalAmount = 0;

        emit PredictionCreated(predictionCount, _question, _closingTime);
        predictionCount++;
    }

    // Function to place a bet on a prediction
    function placeBet(uint256 predictionId) public payable {
        require(block.timestamp < predictions[predictionId].closingTime, "Betting period has ended.");
        require(msg.value > 0, "You must place a positive bet.");

        predictions[predictionId].bets[msg.sender] += msg.value;
        predictions[predictionId].totalAmount += msg.value;

        emit BetPlaced(predictionId, msg.sender, msg.value);
    }

    // Function to resolve the prediction (only callable by the owner)
    function resolvePrediction(uint256 predictionId, uint256 _outcome) public onlyOwner {
        require(block.timestamp >= predictions[predictionId].closingTime, "Prediction is still ongoing.");
        require(predictions[predictionId].outcome == 0, "Prediction has already been resolved.");

        predictions[predictionId].outcome = _outcome;

        emit PredictionResolved(predictionId, _outcome);
    }

    // Function to claim rewards after prediction resolution
    function claimReward(uint256 predictionId) public {
        require(predictions[predictionId].outcome != 0, "Prediction has not been resolved.");
        require(predictions[predictionId].bets[msg.sender] > 0, "You didn't place a bet on this prediction.");

        uint256 betAmount = predictions[predictionId].bets[msg.sender];
        uint256 reward = 0;

        if (predictions[predictionId].outcome == 1) {
            // Winner gets their portion of the total pool
            reward = (betAmount * predictions[predictionId].totalAmount) / address(this).balance;
            payable(msg.sender).transfer(reward);
        }

        // Reset the bet for the player
        predictions[predictionId].bets[msg.sender] = 0;

        emit RewardClaimed(predictionId, msg.sender, reward);
    }

    // Function to view prediction details
    function getPrediction(uint256 predictionId) public view returns (string memory, uint256, uint256, uint256) {
        return (
            predictions[predictionId].question,
            predictions[predictionId].closingTime,
            predictions[predictionId].totalAmount,
            predictions[predictionId].outcome
        );
    }
}
