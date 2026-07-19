+++
date = '2026-07-19T12:29:58+01:00'
draft = false
title = 'RCPL: A C REPL'
+++
# RCPL: A C REPL
---
While tools like [Cling](https://github.com/root-project/cling) exist that implement the Read Eval Print Loop concept, they generally tend to re-invent the wheel by actually *interpreting* the C code. This is why I made [RCPL](https://github.com/ZhaolunYin/rcpl) - Read ***Compile*** Print Loop. Instead of interpreting the C code, RCPL pushes the code to a buffer and then calls *gcc* or *clang* to compile the code.

```bash
$ ./rcpl
>>> printf("Hello, world!\n");
Hello, world!
>>> int i = 123;
>>> printf("i is %d\n", i);
i is 123
>>> print(hi);
.buf_41318.c: In function ‘main’:
.buf_41318.c:16:9: error: implicit declaration of function ‘print’; did you mean ‘printf’? [-Wimplicit-function-declaration]
   16 |         print(hi);
      |         ^~~~~
      |         printf
.buf_41318.c:16:15: error: ‘hi’ undeclared (first use in this function); did you mean ‘i’?
   16 |         print(hi);
      |               ^~
      |               i
.buf_41318.c:16:15: note: each undeclared identifier is reported only once for each function it appears in
>>>
$ 
```

---
## Buffer
To store the user's input, the obvious solution is to use a **file**. However, this can be affected by other programs so it is safer to use a character array stored in the memory. The following function concatentates a second string onto the first string.
```c
// Function similar to fputs except for char *
void buf_add(char **buffer, const char *add) {
    // Allocate memory for buffer + new + null byte
    size_t buf_len = strlen(*buffer);
    size_t add_len = strlen(add);
    char *tmp = realloc(*buffer, (buf_len + add_len + 1) * sizeof(char));
    if (!tmp) {
        perror("realloc");
        exit(1);
    }
    *buffer = tmp;
    // Copy add onto buffer
    memcpy(*buffer + (buf_len * sizeof(char)), add, add_len + 1); 
}
```
The initial buffer has the basic C program structure along with a part which redirects *stdout* to */dev/null*. This is because just before compilation, *stdout* is *restored*, so only what was printed by the *most recent* line of code shows.
```c
// Setup function
char *setup() {
    // make buffer empty heap string 
    char *buffer = malloc(sizeof(char));
    buffer[0] = '\0';
    // add starting file, redirects stdout to /dev/null
    buf_add(&buffer, 
            "#include <stdio.h>\n"
            "#include <stdlib.h>\n"
            "#include <fcntl.h>\n"
            "#include <unistd.h>\n"
            "int main() {\n"
            "\tint _STDOUT, _DEVNULL;\n"
            "\tfflush(stdout);\n"
            "\t_STDOUT = dup(1);\n"
            "\t_DEVNULL = open(\"/dev/null\", O_WRONLY);\n"
            "\tdup2(_DEVNULL, 1);\n"
            "\tclose(_DEVNULL);\n"
            );
    return buffer;
}
```
---
## Compile and Run
To compile and run, the program first copies the current buffer into a *new* character array in order to easily discard changes later. Then, it restores *stdout* and pushes the new line of text onto the buffer. Then, it adds the final curly brace to close the main function. The program then prints the whole buffer into a file, forks a **C compiler** with speed-focused flags to compile it, and runs the program. The files are then deleted and if compilation *and* run were both successful, pushes the line onto the main buffer.
```c
int run(char **buffer, char *cfile, char *xfile, char *text) {
    // Copy bufffer into temporary compile buffer
    char *comp_buf = malloc(sizeof(char) * (strlen(*buffer) + 1));
    if (!comp_buf) {
        perror("malloc");
        exit(1);
    }
    strcpy(comp_buf, *buffer);

    // Enable stdout and add newline + tab
    buf_add(&comp_buf, "\tfflush(stdout); dup2(_STDOUT, 1); close(_STDOUT);\n\t");
    // add line
    buf_add(&comp_buf, text);
    // Add newline and curly brace so it can compile
    buf_add(&comp_buf, "\n}\n");

    // Output comp_buf to a file for compilation
    FILE *compile = fopen(cfile, "w");
    if (!compile) {
        perror("fopen");
        free(comp_buf);
        return -1; 
    }
    fprintf(compile, "%s", comp_buf);
    fclose(compile);

    // Compile
    pid_t pid = fork();
    if (pid == 0) {
#if defined(__clang__)
            char *compiler = "clang";
#elif defined(__GNUC__)
            char *compiler = "gcc";
#else
            char *compiler = "cc";  // unknown compiler, fall back to cc
#endif
        char *args[] = {
            compiler,
            "-O0",
            "-w",
            "-fno-asynchronous-unwind-tables",
            "-fno-ident",
            "-pipe",
            cfile,
            "-o",
            xfile,
            NULL
        };
        execvp(compiler, args);
        perror("execvp");
        exit(1);
    }
    else if (pid < 0) {
        perror("fork");
        free(comp_buf);
        return -1;
    }

    // wait for compilation to finish
    int status;
    waitpid(pid, &status, 0);

    // Run
    if (WIFEXITED(status) && WEXITSTATUS(status) == 0) {
        pid = fork();
        if (pid == 0) {
            char path[512];
            snprintf(path, sizeof(path), "./%s", xfile);
            char *args[] = { path , NULL };
            execvp(path, args);
            perror("execvp");
            exit(1);
        }
        waitpid(pid, &status, 0);
    }

    // Delete created files and free comp_buf
    unlink(cfile);
    unlink(xfile);
    free(comp_buf);

    // Re-insert line without compile necessary stuff if compile was successful
    if (WIFEXITED(status) && WEXITSTATUS(status) == 0) {
        buf_add(buffer, "\t");
        buf_add(buffer, text);
        buf_add(buffer, "\n");
    }

    // If compileed without interrupt, return exit code else return -1
    return WIFEXITED(status) ? WEXITSTATUS(status) : -1;
}
```
---
## Loop
The following main function handles **input**, **nesting** and a few **commands**. It creates process-independent filenames so concurrent processes do not clobber each other's files. The nesting function is quite simplistic and does not account for a lot of things (e.g. escaped quotes).
```c
// Function to calculate curly brace nesting level
int nest_depth(char *text) {
    int depth = 0;
    int quote1 = 0, quote2 = 0;
    for (int i = 0; text[i] != '\0'; i++) {
        int c = text[i];
        if (c == '\'') {
            quote1 = !quote1;
        }
        else if (c == '\"') {
            quote2 = !quote2;
        }
        else if (c == '{' && !quote1 && !quote2) {
            depth++;
        }
        else if (c == '}' && !quote1 && !quote2) {
            depth--;
        }
    }
    return depth;
}

// Function to parse commands
int command(char *command, char *buffer, char *nest_buf) {
    // If exit command was run, free buffers and exit
    if (strcmp(command, ":exit") == 0) {
        free(command);
        free(buffer);
        free(nest_buf);
        exit(0);
    }
    // If file command was run, print file
    else if (strcmp(command, ":file") == 0) {
        printf("%s%s\n}\n", buffer, nest_buf);
    }
    // If no command was run, return 0 (false)
    else {
        return 0;
    }
    // Else return 1 (true)
    return 1;
}

int main() {
    // Setup
    char *buffer = setup();
    if (!buffer) {
        return 1;
    }

    // Create cfile and xfile names unique to process
    char cfile[64], xfile[64];
    snprintf(cfile, sizeof(cfile), ".buf_%d.c", getpid());
    snprintf(xfile, sizeof(xfile), ".run_%d", getpid());
    int nesting = 0;
    // Create empty string nest_buf for storage of nested parts without compilation
    char *nest_buf = malloc(sizeof(char));
    nest_buf[0] = '\0';

    while (1) {
        // Read line from stdin
        char *line = NULL;
        size_t size = 0;
        printf(">>> ");
        fflush(stdout);

        ssize_t nread = getline(&line, &size, stdin);
        if (nread == -1) {
            free(line);
            break;
        }
        if (nread > 0 && line[nread - 1] == '\n') {
            line[nread - 1] = '\0';
        }
        if (command(line, buffer, nest_buf)) {}
        // If user typed exit(n); or return n; then exit.
        else if (strstr(line, "exit(") == line  || strstr(line, "return ") == line) {
            free(buffer);
            free(nest_buf);
            free(line);
            return 0;
        }
        // if not nesting, run line
        else if (nest_depth(buffer) + nest_depth(nest_buf) + nest_depth(line) == 1 && !nesting) {
            run(&buffer, cfile, xfile, line);
        }
        // else if finished nesting run nested part
        else if (nest_depth(buffer) + nest_depth(nest_buf) + nest_depth(line) == 1 && nesting) {
            buf_add(&nest_buf, line);
            buf_add(&nest_buf, "\n");
            nesting = 0;
            run(&buffer, cfile, xfile, nest_buf);
            free(nest_buf);
            nest_buf = malloc(sizeof(char));
            nest_buf[0] = '\0';
        }
        // if there are more '}' than '{'
        else if (nest_depth(buffer) + nest_depth(nest_buf) + nest_depth(line) < 1) {
            free(nest_buf);
            free(buffer);
            printf("Error: unexpected \'}\'\n");
            return 1;
        }
        // else add to nest buf
        else {
            nesting = 1;
            buf_add(&nest_buf, line);
            buf_add(&nest_buf, "\n");
        }
        // free line
        free(line);
    }
    free(nest_buf);
    free(buffer);
    return 0;
}
```

# Conclusion
In conclusion, although tools such as **Cling** are much more advanced and powerful, a simple tool such as this can suffice for occasional testing purposes.
