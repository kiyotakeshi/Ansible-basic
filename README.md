# Ansibleの検証

- ターゲットはGCP上のインスタンス
    - always freeインスタンス + 3つ以上
    - inventoryを毎回書き換えるのが手間なので、staticIPを割り当てる
    - [インスタンス未稼働時に課金は発生(1インスタンス24円/1日)](https://cloud.google.com/compute/all-pricing?hl=ja#ipaddress)

- 実行元はローカルのMac