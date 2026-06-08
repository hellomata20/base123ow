# base123ow
0x9e4E1f2B064a489E48EB9b427fD0f742bdEb9e4C
import time
from collections import defaultdict
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

TRANSFER_TOPIC = Web3.keccak(
    text="Transfer(address,address,uint256)"
).hex()

DEPLOY_LOOKBACK = 20
LAUNCH_WINDOW = 10

LAUNCH_SCORE_THRESHOLD = 5

ZERO = "0x0000000000000000000000000000000000000000"


def decode_address(topic):
    return "0x" + topic.hex()[-40:]


def main():

    w3 = Web3(Web3.HTTPProvider(RPC_URL))

    if not w3.is_connected():
        raise RuntimeError("Cannot connect")

    print("Connected to Base")
    print("Detecting launch participants...\n")

    last_block = w3.eth.block_number

    recent_deploys = {}

    wallet_scores = defaultdict(int)

    while True:

        try:

            current_block = w3.eth.block_number

            if current_block > last_block:

                # find contracts created
                for block_num in range(
                    last_block + 1,
                    current_block + 1
                ):

                    block = w3.eth.get_block(
                        block_num,
                        full_transactions=True
                    )

                    for tx in block.transactions:

                        if tx.to is None:

                            receipt = (
                                w3.eth.get_transaction_receipt(
                                    tx.hash
                                )
                            )

                            if receipt.contractAddress:

                                recent_deploys[
                                    receipt.contractAddress
                                ] = block_num


                logs = w3.eth.get_logs({
                    "fromBlock": last_block + 1,
                    "toBlock": current_block,
                    "topics": [TRANSFER_TOPIC]
                })


                for log in logs:

                    token = log["address"]

                    if token not in recent_deploys:
                        continue

                    age = (
                        log["blockNumber"]
                        - recent_deploys[token]
                    )

                    if age <= LAUNCH_WINDOW:

                        buyer = decode_address(
                            log["topics"][2]
                        )

                        if buyer != ZERO:
                            wallet_scores[buyer] += 1


                print(
                    f"\nBlocks {last_block}"
                    f" -> {current_block}"
                )

                for wallet, score in wallet_scores.items():

                    if score >= LAUNCH_SCORE_THRESHOLD:

                        print("🚀 Launch Participant")
                        print("Wallet:", wallet)
                        print("Launch score:", score)
                        print()

                last_block = current_block


            time.sleep(3)


        except Exception as e:

            print("Error:", e)
            time.sleep(5)


if __name__ == "__main__":
    main()
