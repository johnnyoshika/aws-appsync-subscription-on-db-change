```
aws logs describe-log-groups

aws logs describe-log-streams --log-group-name "get logGroupName from previous command" --order-by LastEventTime
```

The next command will only work if | works (e.g. bash). [jq](https://stedolan.github.io/jq/) will also need to be installed.

You may get a `The specified log stream does not exist.` error. It's probably because the log stream name has a `$` in it. Escape it with `\$` in bash or `` `$ `` in powershell.

```
aws logs get-log-events --log-group-name "get logGroupName from previous command" --log-stream-name "get logStreamName from previous command" | jq '.events[].timestamp |= ( ./ 1000 | strftime("%Y-%m-%d %T%p"))'
```

Note: the above won't work in powershell, so may need to simplify like this:
```
aws logs get-log-events --log-group-name "get logGroupName from previous command" --log-stream-name "get logStreamName from previous command"
```

Or if you install jq via scoop, then this will work (but it's not any better than the last one as I  couldn't format the date nicely)

```
aws logs get-log-events --log-group-name "get logGroupName from previous command" --log-stream-name "get logStreamName from previous command" | jq '.events[].timestamp |= ( ./ 1000 )'
```

