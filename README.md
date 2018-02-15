# Ethereum Voting Framework
A process and tools to enable large scale voting on ethereum. Uses the following technologies:
- An Ethererum client for generating one time use accounts
- Commit/Reveal smart contract pattern for hiding votes while vote is ongoing 

### Vote requirements:
- Voters must be verified
- Voters can only vote once
- A voter's vote cannot be know until election ends
- The association between a voter's identity and the vote cast is known only to election admins
- Voters can verify their vote after election ends
- The calculation of results is performed by the ethereum smart contract

# Process 


## Admin Account Setup
The commit/reveal pattern requires that the votes be submitted to the ethereum blockchain encrypted with a secret phrase only known to the election administrators.  The admins also control an account which has access to change various settins in the voting contract.

A voting administration account and secret phrase is created using an offline process. The recovery phrase and secret phrase are split among n administrators.  The secret phrase is sha3 encoded offline and copied for the ballot preparation step.  It is critical that the secret and recovery phrase are kept secure to ensure the integrity of the voting process.
```
var adminSecret = 'A really secret phrase'
var sha3adminSecret = sha3(adminSecret)
```
## Voting Contract Deployment
The voting contract is deployed to the ethereum network and the contract address is recorded for use in the Ballot Generation step.  Enough ether is sent to the contrat to handle estimated gas costs.  Also the gasPrice and voteGasAmount is set.

## Ballot Generation
The sha3AdminSecret is securely transfered to a new storage device used in the ballot generation process.  The ballot generation process is also performed offline to keep the sha3AdminSecret secret.  If the sha3AdminSecret is revealed during the voting period then the voting results can be determined via a rainbow attack against the ballot choices.

- Each ballot configuration (e.g. one per distict) is converted to javascript object of questions/answers 
``` 
//WIP - not sure if normalizing questions & ballots is a good design choice
questions =  [
  { globalId: 0, title: 'Mayor', answ0ers: ['Alice','Bob','Charlie','Abstain']},
  {globalId: 1, title: 'Amendment 21', answers: ['Yes','No','Abstain']}
  ]

ballot = {id: 'district1', questions: [0,1] choices: [[0,0],[1,0],[2,0],[3,0]...]};
anotherBallot = {id: 'district2', questions: [0]}; --ammendment 21 not included on this ballot

```
- An array is created of all possible combinations
```
ballotChoices = [
'sha3(ballot.id + ':' + ballot.questions[0] + ':' ballot.answers[0] + ',' + ballot.questions[1] + ':' + ballot.answers[0])', --district1:Mayor:Alice,Admendment 21:Yes,
'sha3(ballot.id + ':' + ballot.questions[0] + ':' ballot.answers[1] + ',' + ballot.questions[1] + ':' + ballot.answers[0])', --district1:Mayor:Bob,Admendment 21:Yes,
...repeat for x times where x equals the factorial of each anwer's number of questions (e.g. 4x3 = 12)
]
ballotChoices = [
  '0211000100',
  '0210100100'
]

```
- For each ballot to be assigned to a voter the server runs an ethereum tool to generate and unlock a new account
- Server creates a directory under ballotDirectory/ballotId/userAccount 
- Server creates encrypted submissions for all possible ballot choices using choice, user's Ethereum account and sha3adminSecret, in this example there  would be 12 tx's generated, one for each possible combination of ballot choices
```
  var submission[n] = {sha3(choice): n, encryptedValue: sha3(ballotCoice[n]+ethereumAccount+sha3(adminSecret)}
  e.g. submission[6] = {choice: 0AEFGHSLT...', encryptedValue: sha3('088877..'+'0x83200...'+sha3(adminSecret)} -- 0x83200 votes Bob,No
```
- Server creates etherum tx's from user's account for each encrypted ballot choice all with nonce = 0 (this prevents multiple choices from being used in the voting process for a given account), it then creates a wrapping "forwarding" transaction to use gas from the forwarding contract
```
  tx[n] =  signTx(from: userAccount, to: votingContract, function: vote, valueData: submission[n].encryptedValue,nonce=0)
  userBallot = {account: accountAddr, tx: tx}
```
#### TODO Try to figure out a way to disconnect the identity from the tx casting the vote

## Voter Registration
This is would use conventional methods to setup voters with user/passwords on the voting website.  In addition some form of attestation could be added to verify identification and uniqueness of a voter.

## Ballot Asssignment
- User logs on to the voting administrator's website
- User requests ballot
- Server assignes the the userBallot.account to the user's login id 
- Server registers the userBallot.account on the voting contract and sends enough ether to pay for vote
```
register(address voterAddress) {
  voters[voterAddress] = '0000000';
  voterAddress.send(20000 wei);
  //TODO create address lookup array
  voters++;
}
```
- Server returns the full list of transactions, voter's ethereum address, ballot and ballot choices to user's browser

## Vote Submission
- The voter's web page loads the ballot
- Voter makes selections
- Voter submits the tx to ther ethereum network corresponding to the combinations of ballotChoices
```
web3.sendSignedTx(ballot.tx[i]);
```
- the tx invokes the vote method on the voting contract
```
vote(bytes[] encryptedVote, ballotId) {
  voters[msg.sender].vote = encryptedVote;
  voters[msg.sender].id = ballotId;
  voters[msg.sender].tallied = 0;
}
```
## Voter Verification
- Voter calls voting contract's voters function to verify that the encrypted value matches his ballotChoice (the voters sha3 ballotChoices are only known to him)


## Vote Reveal & Tally
- Election admin submits the adminSecret to the contract
```
reveal(bytes[] adminSecret onlyAdmin{
  secret = adminSecret;
  sha3Secret = sha3(adminSecret)
}
```

- At this point it is possible to iterate through the possible ballot choices and determine the outcome. Anyone can now iterate through the voters array and determine the vote tallies for each ballotChoice
- Admin calls the tally function 
```
tally() static readonly {
  //todo convert to static function (no gas limit?)
  foreach( var i = startIndex; i <endIndex, i++) {
    if (votes[i].tallied) continue;
    var userAccount = votes[i].userAccount;
    var ballotId = votes[i].ballotId;
    //verify sha3
    var sha3Vote = sha3(vote+userAccount+sha3AdminSecret);
    if (sha3Vote != votes[i].encryptedvote) {
      votes[i].tallied = 1;
      continue;
    }
    //validate vote
    if (vote not in validVotesForBallot(ballotId)) 
    {
      votes[i].tallied = 1;
      votes[i].valid = 0;
    }
    //foreach byte = 1 in vote increment questions array for ballot

    votes[i].tallied = 1;
    votes[i].valid = 1;
  }
  
}

```
At this point cross ballot reporting should be trivial and easily verifiable.

//TODO add master questionid to ballot struct
