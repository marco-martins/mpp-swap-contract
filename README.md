# Marlowe Pioneers

Small tutorial showing how to create and execute a Marlowe smart contract using marlow-cli through the Linux shell.
The contract consists of two parties (Party1 and Party2) where each one deposit a specific amount of ADA and then the contract swaps the amounts.
All the steps were executed with a fresh installation with Ubuntu 20.04.4 LTS.

#### Basic Setup

Install NIX

```bash

sudo apt update && sudo apt upgrade -y
sudo apt install curl xz-utils -y

sh <(curl -L https://nixos.org/nix/install) --daemon
nix-shell -p nix-info --run "nix-info -m"
nix-env --version

sudo nano /etc/nix/nix.conf
# add the following content to the end of the file and save #######
substituters = https://hydra.iohk.io https://iohk.cachix.org https://cache.nixos.org/
trusted-public-keys = hydra.iohk.io:f/Ea+s+dFdN+3Y/G+FDgSq+a5NEWhJGzdjvKNGv0/EQ= iohk.cachix.org-1:DpRUyj7h7V830dp/i6Nti+NEO2/nhblbov/8MW7Rqoo= cache.nixos.org-1:6NCHdD59X431o0gWypbMrAURkbJ16ZPMQFGspcDShjY=
experimental-features = nix-command flakes
allow-import-from-derivation = true
######## ctrl+x and save the changes

sudo systemctl restart nix-daemon.service
```

##### Download Marlowe Cardano

```bash
git clone https://github.com/input-output-hk/marlowe-cardano.git
cd marlowe-cardano
git checkout marlowe-pioneers
nix-shell # wait a couple of minutes for the files compile
start-marlowe-run # wait a couple of minutes for the node fully sync
# To exit from the TMUX session press "ctrl+b d"
```

##### Setup envs

```bash
# sudo find / -name '*node.socket'
export CARDANO_NODE_SOCKET_PATH=/tmp/node.socket
export CARDANO_TESTNET_MAGIC=1567
```

**_*Note*_** `CARDANO_TESTNET_MAGIC=1097911063` for the public testnet. Replace `--testnet-magic` to `--mainnet` for the mainnet.

#### Create wallets

The `wallet1` will belong to the `Party1` and the `wallet2` will belong to the `Party2`.
Generate the seed phrase `mnemonic` and convert it to an extended private key file `wallet.prv`, note that we are also saving the file `wallet.seed`.

```bash
mkdir -p wallets/wallet1
cd wallets/wallet1
```

```bash
# Generate the seed phrase and convert it into a root private key `wallet.prv`
cardano-wallet recovery-phrase generate --size 15 \
| tee wallet.seed \
| cardano-wallet key from-recovery-phrase Shelley \
| cardano-wallet key child 1852H/1815H/0H/0/0 > wallet.prv

# Generate the wallet files from the wallet.priv file previews generated
cardano-cli key convert-cardano-address-key --shelley-payment-key --signing-key-file wallet.prv --out-file payment.skey
cardano-cli key verification-key --signing-key-file payment.skey --verification-key-file payment.vkey
cardano-cli address build --testnet-magic 1567 --payment-verification-key-file payment.vkey > payment.addr
```

**_Note_** Repeat the previous steps to create the `wallet2`, the wallet folder should be named `wallet2`.

##### Funding the wallet

```bash
marlowe-cli util faucet \
--testnet-magic $CARDANO_TESTNET_MAGIC \
--lovelace 2000000000 \
--out-file /dev/null \
--submit 600 $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)
```

**_Note_** this only work for the testnet 1567 (Marlowe Pioneers)

##### Query the balance of an address

```bash
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC
```

#### Smart Contract

Setup variables

```bash
MILLISECOND=1000
NOW=$(($(date -u +%s)*MILLISECOND))  # The current time in POSIX milliseconds.
HOUR=$((60*60*MILLISECOND))          # One hour, in POSIX milliseconds.
DEADLINE=$((NOW+10*HOUR))            # Timeout deadline, ten hours from now.
MINIMUM_ADA=2000000

PARTY_1=Party1
PARTY_2=Party2
PARTY_1_AMOUNT=100000000
PARTY_2_AMOUNT=200000000
```

##### Mint Role Tokens

```bash
cd ~/marlowe-cardano/
mkdir my-contract && cd my-contract

ROLES_CURRENCY=$(marlowe-cli util mint \
  --required-signer ~/marlowe-cardano/wallets/wallet1/payment.skey \
  --change-address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
  --out-file asset \
  --submit 600 \
  "$PARTY_1" "$PARTY_2" \
| sed -e 's/^PolicyID "\(.*\)"$/\1/')

echo "Roles currency $ROLES_CURRENCY"
echo "The hexadecimal representation of $PARTY_1 is $ROLES_CURRENCY.$(echo -n $PARTY_1 | basenc --base16)."
echo "The hexadecimal representation of $PARTY_2 is $ROLES_CURRENCY.$(echo -n $PARTY_2 | basenc --base16)."
```

##### Query the address to see that the tokens have been minted

```bash
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC

# Output
                           TxHash                                 TxIx        Amount
--------------------------------------------------------------------------------------
b437bd3fb73ffe4c87ec0ecdd73d5adb3fcbb8df6933d63d5a661e695436fed2     0        1979823763 lovelace + TxOutDatumNone
b437bd3fb73ffe4c87ec0ecdd73d5adb3fcbb8df6933d63d5a661e695436fed2     1        10000000 lovelace + 1 e6ab256a5a428c4427299149fa3ea6e055a966f7ddba7f8033841c3f.506172747931 + TxOutDatumNone
b437bd3fb73ffe4c87ec0ecdd73d5adb3fcbb8df6933d63d5a661e695436fed2     2        10000000 lovelace + 1 e6ab256a5a428c4427299149fa3ea6e055a966f7ddba7f8033841c3f.506172747932 + TxOutDatumNone

# The TxIx 0 is where we have our remaining ADA
# The TxIn 1 is where we have the Party1 role token
# The TxIn 2 is where we have the Party2 role token
```

**_*Note*_** To make the next transactions we need the make reference to this `TxHash` and `TxIx`.

##### Send funds to the second wallet

Let's send to the Party 2 the role token and also 1000 ADA `wallet2`.

```bash
# --tx-in TxHash#TxId

marlowe-cli transaction simple \
--required-signer ~/marlowe-cardano/wallets/wallet1/payment.skey \
--tx-in b437bd3fb73ffe4c87ec0ecdd73d5adb3fcbb8df6933d63d5a661e695436fed2#0 \
--tx-in b437bd3fb73ffe4c87ec0ecdd73d5adb3fcbb8df6933d63d5a661e695436fed2#1 \
--tx-in b437bd3fb73ffe4c87ec0ecdd73d5adb3fcbb8df6933d63d5a661e695436fed2#2 \
--change-address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)+$MINIMUM_ADA+1 $ROLES_CURRENCY.$PARTY_1" \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet2/payment.addr)+$MINIMUM_ADA+1 $ROLES_CURRENCY.$PARTY_2" \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet2/payment.addr)+1000000000" \
--out-file /dev/null \
--print-stats \
--submit 600
```

##### Construct the contract from a template

```bash
# Do not run
marlowe-cli template simple \
--minimum-ada "$MINIMUM_ADA" \
--bystander "Role=$PARTY_2" \
--party "Role=$PARTY_1" \
--deposit-lovelace "$PARTY_2_AMOUNT" \
--withdrawal-lovelace "$PARTY_2_AMOUNT" \
--timeout "$DEADLINE" \
--out-contract-file contract.json \
--out-state-file tx1.state
```

**_*Note*_** Not applicable here because we gonna test with our own contract (not from the templates). In case you use a contract from the templates, the command above automatically generates the `contract.json` and the `tx1.state` files for you.

##### Generate contract.json file

You can create your own contract using marlowe-playground, export it as JSON and then adjust the variables according.

```bash
cat > contract.json << EOI
{
  "when": [
    {
      "then": {
        "when": [
          {
            "then": {
              "token": {
                "token_name": "",
                "currency_symbol": ""
              },
              "to": {
                "party": {
                  "role_token": $PARTY_2
                }
              },
              "then": {
                "token": {
                  "token_name": "",
                  "currency_symbol": ""
                },
                "to": {
                  "party": {
                    "role_token": $PARTY_1
                  }
                },
                "then": "close",
                "pay": $PARTY_2_AMOUNT,
                "from_account": {
                  "role_token": $PARTY_2
                }
              },
              "pay": $PARTY_1_AMOUNT,
              "from_account": {
                "role_token": $PARTY_1
              }
            },
            "case": {
              "party": {
                "role_token": $PARTY_2
              },
              "of_token": {
                "token_name": "",
                "currency_symbol": ""
              },
              "into_account": {
                "role_token": $PARTY_2
              },
              "deposits": $PARTY_2_AMOUNT
            }
          }
        ],
        "timeout_continuation": "close",
        "timeout": $(( NOW + 12 * HOUR ))
      },
      "case": {
        "party": {
          "role_token": $PARTY_1
        },
        "of_token": {
          "token_name": "",
          "currency_symbol": ""
        },
        "into_account": {
          "role_token": $PARTY_1
        },
        "deposits": $PARTY_1_AMOUNT
      }
    }
  ],
  "timeout_continuation": "close",
  "timeout": $(( NOW + 10 * HOUR ))
}
EOI
```

##### Generate tx1.state (inicial state of contract)

```bash
cat > tx1.state << EOI
{
  "accounts": [
    [[{ "role_token": "$PARTY_1"}, { "currency_symbol": "", "token_name": "" }], $MINIMUM_ADA]
  ],
  "choices": [],
  "boundValues": [],
  "minTime": 1
}
EOI
```

##### Initialize the contract

If you wanna just test the contract, you can run only the `simulation`commands.
If you wanna run the contract on-chain you need to run the `simulation` and the `execution` commands.

```bash
marlowe-cli run initialize \
--roles-currency "$ROLES_CURRENCY" \
--contract-file contract.json \
--state-file tx1.state \
--out-file tx1.marlowe \
--print-stats
```

##### Execution of the inicial deposit

```bash

cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC
```

```bash
# tx-in TX_PARTY_1_ADA

marlowe-cli run execute \
--testnet-magic $CARDANO_TESTNET_MAGIC \
--tx-in "cc4dd70bc593b3584fa2e60bcd5d5025d9d7888734568864822a6d918d39c513#0" \
--change-address "$(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)" \
--required-signer ~/marlowe-cardano/wallets/wallet1/payment.skey \
--marlowe-out-file tx1.marlowe \
--out-file /dev/null \
--print-stats \
--submit 600
```

```bash
jq '.contract' tx1.marlowe | yq -y
CONTRACT_ADDRESS=$(jq -r '.marloweValidator.address' tx1.marlowe)
cardano-cli query utxo --address "$CONTRACT_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

##### Simulation transaction 1

```bash
marlowe-cli run prepare \
--marlowe-file tx1.marlowe \
--deposit-account "Role=$PARTY_1" \
--deposit-party "Role=$PARTY_1" \
--deposit-amount "$PARTY_1_AMOUNT" \
--invalid-before "$NOW" \
--invalid-hereafter "$((NOW+9*HOUR))" \
--out-file tx2.marlowe \
--print-stats

jq '.contract' tx2.marlowe | yq -y
```

**_Note_** the perpare command move your contract from one stage to another stage.

##### Execution transaction 1

```bash
# Party 1 address
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC

# Contract adresss
cardano-cli query utxo --address "$CONTRACT_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

```bash
# --tx-in-marlowe TX_CONTRACT
# --tx-in-collateral TX_PARTY_1_ADA
# --tx-in TX_PARTY_1_ADA
# --tx-in TX_PARTY_1_TOKEN

marlowe-cli run execute \
--marlowe-in-file tx1.marlowe \
--tx-in-marlowe 59fd8c323155f0e6c2c7a0b679722b659c1d0d493c1d590f09f03c3561443dcf#1 \
--tx-in-collateral 59fd8c323155f0e6c2c7a0b679722b659c1d0d493c1d590f09f03c3561443dcf#0 \
--tx-in 59fd8c323155f0e6c2c7a0b679722b659c1d0d493c1d590f09f03c3561443dcf#0 \
--tx-in cc4dd70bc593b3584fa2e60bcd5d5025d9d7888734568864822a6d918d39c513#1 \
--required-signer ~/marlowe-cardano/wallets/wallet1/payment.skey \
--marlowe-out-file tx2.marlowe \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)+$MINIMUM_ADA+1 $ROLES_CURRENCY.$PARTY_1" \
--change-address "$(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)" \
--out-file /dev/null \
--print-stats \
--submit 600
```

```bash
jq '.contract' tx2.marlowe | yq -y
cardano-cli query utxo --address "$CONTRACT_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

##### Simulation transaction 2

```bash
marlowe-cli run prepare \
--marlowe-file tx2.marlowe \
--deposit-account "Role=$PARTY_2" \
--deposit-party "Role=$PARTY_2" \
--deposit-amount "$PARTY_2_AMOUNT" \
--invalid-before "$NOW" \
--invalid-hereafter "$((NOW+9*HOUR))" \
--out-file tx3.marlowe \
--print-stats

# Datum size: 23
# Payment 1
#   Acccount: "Party1"
#   Payee: Party "Party2"
#   Ada: 100.000000
# Payment 2
#   Acccount: "Party2"
#   Payee: Party "Party1"
#   Ada: 200.000000
# Payment 3
#   Acccount: "Party1"
#   Payee: Party "Party1"
#   Ada: 2.000000

jq '.contract' tx3.marlowe | yq -y
```

##### Execution transaction 2

```bash
# Party 1 address
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet2/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC

# Contract address
cardano-cli query utxo --address "$CONTRACT_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

```bash
# --tx-in-marlowe TX_CONTRACT
# --tx-in-collateral TX_PARTY_2_ADA
# --tx-in TX_PARTY_2_ADA
# --tx-in TX_PARTY_2_TOKEN

marlowe-cli run execute \
--marlowe-in-file tx2.marlowe \
--tx-in-marlowe bca133ed3e4e39601481cfafde4d46a499eb47010a3c07cff779e7365f297990#1 \
--tx-in-collateral cc4dd70bc593b3584fa2e60bcd5d5025d9d7888734568864822a6d918d39c513#3 \
--tx-in cc4dd70bc593b3584fa2e60bcd5d5025d9d7888734568864822a6d918d39c513#3 \
--tx-in cc4dd70bc593b3584fa2e60bcd5d5025d9d7888734568864822a6d918d39c513#2 \
--required-signer ~/marlowe-cardano/wallets/wallet2/payment.skey \
--marlowe-out-file tx3.marlowe \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet2/payment.addr)+$MINIMUM_ADA+1 $ROLES_CURRENCY.$PARTY_2" \
--change-address "$(cat ~/marlowe-cardano/wallets/wallet2/payment.addr)" \
--out-file /dev/null \
--print-stats \
--submit 600
```

```bash
jq '.contract' tx2.marlowe | yq -y
cardano-cli query utxo --address "$CONTRACT_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

**_*Note*_** the contract ends, but the parties need to withdraw their funds.

##### Party 1 withdraw their funds from the payout script address

```bash
# Party 1 address
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC

# Contract payout address
ROLE_ADDRESS=$(jq -r '.rolesValidator.address' tx1.marlowe)
echo $ROLE_ADDRESS
cardano-cli query utxo --address "$ROLE_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

```bash
# --tx-in TX_PARTY_1_ADA
# --tx-in-collateral TX_PARTY_1_ADA
# --tx-in TX_PARTY_1_TOKEN

marlowe-cli run withdraw \
--marlowe-file tx3.marlowe \
--role-name $PARTY_1 \
--tx-in bca133ed3e4e39601481cfafde4d46a499eb47010a3c07cff779e7365f297990#0 \
--tx-in-collateral bca133ed3e4e39601481cfafde4d46a499eb47010a3c07cff779e7365f297990#0 \
--tx-in bca133ed3e4e39601481cfafde4d46a499eb47010a3c07cff779e7365f297990#2 \
--required-signer ~/marlowe-cardano/wallets/wallet1/payment.skey \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)+$MINIMUM_ADA+1 $ROLES_CURRENCY.$PARTY_1" \
--change-address "$(cat ~/marlowe-cardano/wallets/wallet1/payment.addr)" \
--out-file /dev/null \
--print-stats \
--submit 600
```

```bash
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC
```

**_Note_** the PARTY_1 also received 2 ADA, it's the value of the initial deposit to initialize the contract.

##### Party 2 withdraw their funds from the payout script address

```bash
# Party 2 address
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet2/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC

# Contract payout address
cardano-cli query utxo --address "$ROLE_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

```bash
# --tx-in TX_PARTY_2_ADA
# --tx-in-collateral TX_PARTY_2_ADA
# --tx-in TX_PARTY_2_TOKEN

marlowe-cli run withdraw \
--marlowe-file tx3.marlowe \
--role-name $PARTY_2 \
--tx-in cfee29d8b65d6b3ef9b4f4495cdc68756d36606ca77c46afc6d6e295152a965b#0 \
--tx-in-collateral cfee29d8b65d6b3ef9b4f4495cdc68756d36606ca77c46afc6d6e295152a965b#0 \
--tx-in cfee29d8b65d6b3ef9b4f4495cdc68756d36606ca77c46afc6d6e295152a965b#3 \
--required-signer ~/marlowe-cardano/wallets/wallet2/payment.skey \
--tx-out "$(cat ~/marlowe-cardano/wallets/wallet2/payment.addr)+$MINIMUM_ADA+1 $ROLES_CURRENCY.$PARTY_2" \
--change-address "$(cat ~/marlowe-cardano/wallets/wallet2/payment.addr)" \
--out-file /dev/null \
--print-stats \
--submit 600

```

```bash
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet2/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC
```

```bash
cardano-cli query utxo --address "$ROLE_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC
```

##### Final balances

```bash
# Contract address
cardano-cli query utxo --address "$CONTRACT_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC

# Contract payout address
cardano-cli query utxo --address "$ROLE_ADDRESS" --testnet-magic $CARDANO_TESTNET_MAGIC

# Wallet 1
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet1/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC

# Wallet 2
cardano-cli query utxo \
--address $(cat ~/marlowe-cardano/wallets/wallet2/payment.addr) \
--testnet-magic $CARDANO_TESTNET_MAGIC
```

**_Note_** the contract payout address is now empty.

---

@ **_Marco Martins [HYPE] Staking pool_**
