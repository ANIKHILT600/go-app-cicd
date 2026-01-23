# Implementation steps

**1. Local App test**:

**a.** Get the app code from developer team and clone/copy locally.

go-web-app/--
            - static/
            - go-web-app.md
            - go.mod     
            - main

Generally you will get above code from developer, Where:

- The go.mod file declares the module path and tracks the project's dependencies. 

If you don't get it, then initialize your Go module in your local terminal:
```
go mod init github.com/<git-username>/<git-repo>
```
example:
```
go mod init github.com/nikhil600/go-app-cicd
```

- "main" is the compiled binary. The below command is used to build the Go application and compile it into a binary executable file
```
go build -o main .
```
Here's what each part does:

go build: This is the command to compile a Go program.

-o main: This flag specifies the output file name for the compiled binary. In this case, the executable will be named main. If you chose another name like "app-binary," the binary would be created with that name.

- Once the main binary is created, it can be executed using ./main to run the application in local to test it.

