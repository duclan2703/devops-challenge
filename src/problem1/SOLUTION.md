# Problem 1 – Solution

**Objective:** Submit HTTP GET requests to `https://example.com/api/:order_id` for every order ID that is selling TSLA, and write all responses to `./output.txt`.

## Single CLI command

```bash
jq -r 'select(.symbol=="TSLA" and .side=="sell") | .order_id' ./transaction-log.txt | xargs -I {} curl -s "https://example.com/api/{}" > ./output.txt
```

## What it does

1. **`jq -r 'select(.symbol=="TSLA" and .side=="sell") | .order_id' ./transaction-log.txt`**  
   Reads each JSON line, keeps only rows with `symbol == "TSLA"` and `side == "sell"`, and prints the `order_id` (one per line)

2. **`xargs -I {} curl -s "https://example.com/api/{}"`**  
   For each of those order IDs, runs `curl -s "https://example.com/api/<order_id>"`. The `-s` flag keeps curl silent

and writes the concatenated response bodies to `./output.txt`.
