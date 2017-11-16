# Ethereum Voting Framework
A process and tools to enable large scale voting on ethereum. Uses the following technologies:
- An Ethererum client for generating one time use accounts
- Commit/Reveal smart contract pattern for hiding votes while vote is ongoing 

# Process 

## Setup
- Ballot is converted to javascript object of questions/answers 
``` 
ballot = [{ title: 'Mayor', answers: ['Alice','Bob','Charlie','None']}, {title: 'Amendment 21', answers: ['Yes','No','None']}]
```
- An array is created of all possible combinations
```
ballotChoices = [
'00', --Alice,Yes
'10', --Bob,Yes
'20', --Charlie,Yes
'30', --None,Yes
'01', --Alice,No
'11',
'21',
'31',
'02',
'12',
'22',
'32'--None,None
]
```
- A voting administration secret is created
```
var adminSecret = 'A really secret phrase'
```
## Voter Registration
- This is would use conventional methods to setup voters with user/passwords on the voting website

## Ballot Generation
- User logs on to the voting administrator's website
- User requests ballot
- Server runs an ethereum tool to generate and unlock a new account
- Server associates the ethereum account with user's id
- Server creates encrypted submissions for all possible ballot choices using choice, user's Ethereum account and adminSecret
```
  var submission[n] = {choice: n, encryptedValue: sha3(ballotCoice[n]+ethereumAccount+adminSecret)}
  e.g. submission[6] = {choice: '11', encryptedValue: sha3('11'+'0x83200...'+adminSecret)} -- 0x83200 votes Bob,No
```
- Server creates etherum tx's from user's account for each encrypted ballot choice all with nonce = 0
```
  tx[n] =  signTx(submission[n].encryptedValue,nonce=0)
```
- Server registers the user's ethereum address on the voting contract and sends enough ether to pay for vote
```
register(address voterAddress) {
  voters[voterAddress] = '0000000';
  voterAddress.send(20000 wei);
  ballots++;
}
```
- Server returns the full list of transactions, voter's ethereum address, ballot and ballot choices to user's browser

## Vote Submission
- The voter's web page loads the ballot
- Voter makes selections
- Voter submits the tx to ther ethereum network corresponding to the combinations of ballotChoices
```
web3.sendSignedTx(tx[i]);
```
- the tx invokes the vote method on the voting contract
```
vote(bytes[] encryptedVote) {
  voters[msg.sender] = encryptedVote;
}
```
## Voter Verification
- Voter calls voting contract's voters function to verify that the encrypted value matches his ballotChoice


## Vote Reveal & Tally
- Election admin submits the adminSecret to the contract
```
reveal(bytes[] adminSecret {
  secret = adminSecret;
}
```
- Anyone can now iterate through the voters array and determin the vote tallies for each ballotChoice
```


```

