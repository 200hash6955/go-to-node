# go-to-node

A simple API to inter-process communicating between Go and NodeJS.

## Quick start

### Go to Node

`main.go`:

```golang
package main

import (
	"fmt"
	"os"
	"os/exec"

	"github.com/200hash6955/go-to-node"
)

func main() {
	cmd := exec.Command("node", "child.js")
	cmd.Stdout = os.Stdout
	cmd.Stderr = os.Stderr

	channel, err := go2node.ExecNode(cmd)
	if err != nil {
		panic(err)
	}
	defer cmd.Process.Kill()

	// Node will output: {hello: "node"}
	channel.Write(&go2node.NodeMessage{
		Message: []byte(`{"hello": "node"}`),
	})

	// Golang will output: {"hello":"golang"}
	msg, err := channel.Read()
	if err != nil {
		panic(err)
	}
	fmt.Println(string(msg.Message))

	// Wait node child process exit
	cmd.Process.Wait()
}
```

`child.js`:

```js
process.on('message', function (msg, handle) {
    console.log(msg);
    process.exit(0);
  });
process.send({hello: 'golang'});
```

Run `go run main.go` to test.

**Output**:

```
{"hello":"golang"}
{ hello: 'node' }
```

### Node to Golang

`main.js`:

```node
const child_process = require('child_process');

let child = child_process.spawn('go', ['run', 'gochild.go'], {
  stdio: [0, 1, 2, 'ipc']
});

child.on('close', (code) => {
  process.exit(code);
});

child.send({hello: "child"});
child.on('message', function(msg, handle) {
  if(msg.hello === "parent") {
    console.log(msg);
    process.exit(0);
  }
  process.exit(1);
});
```

`gochild.go`:

```golang
package main

import (
	"fmt"

	"github.com/200hash6955/go-to-node"
)

func main() {
	channel, err := go2node.RunAsNodeChild()
	if err != nil {
		panic(err)
	}

	// Golang will output: {"hello":"child"}
	msg, err := channel.Read()
	if err != nil {
		panic(err)
	}
	fmt.Println(string(msg.Message))

	// Node will output: {"hello":'parent'}
	err = channel.Write(&go2node.NodeMessage{
		Message: []byte(`{"hello":"parent"}`),
	})
	if err != nil {
		panic(err)
	}
}
```

Run `node ./main.js` to test.

**Output**:

```
{"hello":"child"}
{ hello: 'parent' }
```
