from web3 import Web3
from eth_account import Account
import time
import os
import random

from data_bridge import data_bridge
from keys_and_addresses import private_keys, labels
from network_config import networks

def center_text(text):
    terminal_width = os.get_terminal_size().columns
    return "\n".join([line.center(terminal_width) for line in text.splitlines()])

def clear_terminal():
    os.system('cls' if os.name == 'nt' else 'clear')

description = """
自动桥接机器人  https://bridge.t1rn.io/
操你麻痹Rambeboy,偷私钥🐶
"""

chain_symbols = {
    'Base': '\033[34m',
    'OP Sepolia': '\033[91m',
}

green_color = '\033[92m'
reset_color = '\033[0m'
menu_color = '\033[95m'

explorer_urls = {
    'Base': 'https://sepolia.base.org', 
    'OP Sepolia': 'https://sepolia-optimism.etherscan.io/tx/',
    'BRN': 'https://brn.explorer.caldera.xyz/tx/'
}

def get_balance(web3, my_address):
    return web3.from_wei(web3.eth.get_balance(my_address), 'ether')

def ensure_connection(rpc_url):
    web3 = Web3(Web3.HTTPProvider(rpc_url))
    while not web3.isConnected():
        print(f"Attempting to reconnect to {rpc_url}...")
        time.sleep(5)
    return web3

def send_transaction(web3, account, data, network_name):
    contract_address = networks[network_name]['contract_address']
    nonce = web3.eth.getTransactionCount(account.address, 'pending')
    value_in_wei = web3.toWei(0.1, 'ether')
    gas_estimate = web3.eth.estimateGas({
        'to': contract_address, 'from': account.address, 'data': data, 'value': value_in_wei
    })
    gas_limit = gas_estimate + 50000
    transaction = {
        'nonce': nonce,
        'to': contract_address,
        'value': value_in_wei,
        'gas': gas_limit,
        'maxFeePerGas': web3.eth.get_block('latest')['baseFeePerGas'] + web3.toWei(5, 'gwei'),
        'maxPriorityFeePerGas': web3.toWei(5, 'gwei'),
        'chainId': networks[network_name]['chain_id'],
        'data': data
    }
    signed_txn = web3.eth.account.sign_transaction(transaction, account.key)
    return web3.eth.send_raw_transaction(signed_txn.raw_transaction)

def process_transactions(network_name):
    web3 = ensure_connection(networks[network_name]['rpc_url'])
    print(f"Connected to {network_name}")

    successful_txs = 0
    for bridge in ["Base - OP Sepolia", "OP - Base"]:
        for private_key in private_keys:
            account = Account.from_key(private_key)
            data = data_bridge.get(bridge)
            if data:
                tx_hash = send_transaction(web3, account, data, network_name)
                print_transaction_details(web3, account, tx_hash, network_name)
                successful_txs += 1
                time.sleep(random.uniform(10, 15))
    return successful_txs

def print_transaction_details(web3, account, tx_hash, network_name):
    tx_receipt = web3.eth.wait_for_transaction_receipt(tx_hash)
    balance = get_balance(web3, account.address)
    print(f"{green_color}Transaction sent from: {account.address}")
    print(f"Gas used: {tx_receipt.gasUsed}")
    print(f"Block number: {tx_receipt.blockNumber}")
    print(f"ETH balance: {balance} ETH")
    print(f"Transaction details: {explorer_urls[network_name]}{tx_hash.hex()}\n{reset_color}")

def main():
    print(center_text(description))
    current_network = 'Base'
    successful_txs = process_transactions(current_network)
    print(f"Total successful transactions: {successful_txs}")

if __name__ == "__main__":
    main()
