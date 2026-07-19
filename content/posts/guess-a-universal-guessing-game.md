+++
date = '2026-07-19T14:36:59+01:00'
draft = false
title = 'Guess - A Universal Guessing Game'
+++

# Guess - A Universal Guessing Game
The program [**Guess**](https://github.com/ZhaolunYin/guess) implements a variation of [**Guess Who**](https://en.wikipedia.org/wiki/Guess_Who%3F). The game goes like this:
```pseudocode
CPU selects random item in category
while player does not guess it
    ask player for input
    if player asked for trait (e.g. height)
        print value (e.g. 175cm)
    else if player guessed correctly
        print You guessed correctly
        end
```
The game also has a show all mode where all the traits are shown already and the player only has to guess the item.

---

## Linked lists
I decided to store the values and option names in 2 **linked lists** because they were of variable length. The options are read from a csv file with a header row in this format:

```csv
Name,Color,Taste
Apple,red,sweet
Orange,orange,sour
Grape,purple,tart
```
---
# Implementation
The **node_alloc()** function, returning a pointer to a new node first allocates space for the node itself (2 pointers) and then allocates enough space for the data string to be copied over.

The **parse_line()** function reads over a line of the file, calling node_alloc for every instance of **sepchar** and at the end. It returns the number of options read.

The **cleanup()** function frees all the nodes in the linked list for the header and secret row.

---
```c
struct Node {
    char *data;
    struct Node *next;
};

// Function to create a new node
struct Node *node_alloc(char *data) {
    // Alocate space for node (16 bytes - 8 for char * and 8 for struct Node *)
    struct Node *temp = malloc(sizeof(struct Node));
    // Set next node to null
    temp->next = NULL;
    // Allocate space for text
    temp->data = malloc(sizeof(char) * (strlen(data) + 1)); // + 1 for null byte at end of string
    // Copy text and return pointer to node
    strcpy(temp->data, data);
    return temp;
}

// Function to parse line of csv-style file
int parse_line(struct Node *head, FILE *file, char sepchar) {
    // Init variables
    int c; // Temporary char storage
    int nops = 1; // Number of options - 1 because last option does not end with trailing comma
    int i = 0; // Records place in buffer
    char buf[MAXOPL]; // Buffer to store text before new node is created
    // Set current node to head node for node creation
    struct Node *current = head;
    // Loop over line
    while ((c = getc(file)) != '\n' && c != EOF) {
        // If char is delimiter (new option)
        if (c == sepchar) {
            // End buffer string with null byte
            buf[i] = '\0';
            // Allocate new node for the option
            current->next = node_alloc(buf);
            // Set current optin to the new node
            current = current->next;
            // Go back to the start of the buffer
            i = 0;
            // Increase no. of ops
            nops++;
            // trim spaces before new option
            while((c = getc(file)) == ' ');
            ungetc(c, file);
        }
        else {
            // Push current char to buffer
            if (i < MAXOPL - 1) { // MAXOPL - 1 because null byte is pushed
                buf[i++] = c;
            }
            else {
                buf[MAXOPL - 1] = '\0';
                printf("Length of value exceeds length of buffer. Value truncated at '%s'\n", buf);
            }
        }
    }
    // Push last option with no trailing comma to new node
    buf[i] = '\0';
    current->next = node_alloc(buf);
    current = current->next;
    // Return no. of ops
    return nops;
}

// Function to free all nodes
void cleanup(struct Node *ops, struct Node *secret) {
    for (struct Node *p = ops, *q; p; p = q) {
        free(p->data);
        q = p->next;
        free(p);
    }
    for (struct Node *p = secret, *q; p; p = q) {
        free(p->data);
        q = p->next;
        free(p);
    }
}
```
---
## Game loop
To choose the secret line, first, the lines are counted, then a random number between 1 and the number of lines - 1 and the **getc()** cursor is moved there.

**display_options()** loops over the options and the values, excludint the 1st one (name) showing only the options if show_all is not selected and showing the option and value if it is selected.

**guess_loop()** repeatedly takes guesses from the player and evaluates them - if **strtol()** succeeds, the player is asking for a trait and if it fails, the player is taking a guess. There are also options for case-sensitive guessing and guessing with limited attempts.

```c
// Function to choose random or set line from file (not header)
// by positioning the cursor
int choose_line(FILE *file, int setline) {
    // Move cursor to start
    fseek(file, 0, SEEK_SET);

    // Count lines by counting number of newline chars
    int c;
    int lines = 0;
    while ((c = getc(file)) != EOF) {
        if (c == '\n') {
            lines++;
        }
    }

    // Move cursor to start
    fseek(file, 0, SEEK_SET);

    // needs at least 2 lines - 1 header and 1 data row
    if (lines < 2) {
        fprintf(stderr, "Error: file has no data rows\n");
        exit(1);
    }

    // Choose random value between 0 and lines
    // choose random value, then clamp it between 0 and lines - 1
    // this means the min is 0 and max lines - 2,
    // making the final range is 0 to lines - 1
    // having lines as the max value is unwanted as then it moves the 
    // cursor to the final newline, with no values after
    int rng = (rand() % (lines - 1)) + 1;

    // If set line was chosen, use that
    if (setline) {
        if (setline > 0 && setline < lines) {
            rng = setline;
        }
        else {
            printf("Line %d is not in file. Using random line.\n", setline);
        }
    }

    // Move cursor forward by rng lines
    lines = 0;
    while ((c = getc(file)) != EOF) {
        if (c == '\n') {
            lines++;
        }
        if (lines == rng) {
            break;
        }
    }

    // Return line chosen - cursor has been positioned
    return lines;
}

// Function to print the options
void display_options(struct Node *ops, struct Node *secret, int show_all) {
    int i = 0;
    // For all options and corresponding secrets
    for (struct Node *p = secret->next->next, *o = ops->next->next; p && o; p = p->next, o = o->next) {
        if (show_all) {
            // If show all mode is on, show option: value
            printf("\e[0;93m%s: %s\e[0m\n", o->data, p->data);
        }
        else {
            // else increment i by 1 and print i. option
            printf("%d. %s\n", ++i, o->data);
        }
    }
    // If show_all then add special option to quit else just type 0 to quit
    if (show_all) {
        printf("\e[1;91mType 'quit' to quit.\e[0m\n");
    }
    else {
        printf("0. quit\n");
    }
    printf("\e[1;94m=> ");
}

// Function to repeatedly take guesses from the user
void guess_loop(struct Node *ops, struct Node *secret, int nops, int show_all, int use_case, int max_attempts) {
    // Set no of attempts and boolean value guessed to 0
    int guessed = 0;
    int attempts = 0;
    // While not guessed and no. of attempts does not EQUAL max attempts
    // This is because by default, max_attempts is -1 so if no max attempts is set, it will go forever
    while (!guessed && attempts++ != max_attempts) {
        // Display options
        display_options(ops, secret, show_all);

        // Create input buffer for user
        char input[MAXOPL];
        // Read input from stdin
        if (!fgets(input, MAXOPL, stdin)) {
            printf("\n");
            return;
        }
        input[strcspn(input, "\n")] = '\0';
        // Color is blue because ansi escape code from display_options still persists so reset it
        printf("\e[0m");
        // Try to convert input to int
        char *end;
        long ask = strtol(input, &end, 10);
        // If succes, parse as selected option
        if (*end == '\0' && !show_all) {
            // if valid option
            if (ask < nops && ask >= 0) {
                // set node pointers p and o to the 1st value, because secret and ops are dummy nodes
                struct Node *p = secret->next;
                struct Node *o = ops->next;
                // find the next node ask times (find the (ask)th node)
                for (int k = 0; k < ask; k++) {
                    p = p->next;
                    o = o->next;
                }
                // If the user quit, print the answer and exit
                if (ask == 0) {
                    printf("\e[0;91m");
                    printf("answer: %s\n\n", p->data);
                    printf("\e[0m");
                    guessed = 1;
                }
                // Else, print the corresponding value
                else {
                    printf("\e[0;93m%s: %s\e[0m\n\n", o->data, p->data);
                }
            }
            else {
                // if the user entered an invalid option, 
                // do not count the attempt as a valid attempt
                printf("\e[1;91mInvalid Option\e[0m\n\n");
                attempts--;
            }
        }
        // Else, user must have taken a guess
        else {
            // If they chose to quit
            if (strcmp(input, "quit") == 0) {
                printf("\e[0;91m");
                printf("answer: %s\n\n", secret->next->data);
                printf("\e[0m");
                guessed = 1;
            }
            // else compare to answer
            else if ((use_case ? strcmp(input, secret->next->data) : strcasecmp(input, secret->next->data)) == 0) {
                printf("\e[1;92mCorrect!\nYou guessed it in %d attempts!\e[0m\n", attempts);
                guessed = 1;
            }
            else {
                printf("\e[1;91mIncorrect\e[0m\n\n");
            }
        }
    }
    // if they ran out of attempts
    if (attempts == max_attempts) {
        printf("\e[1;91mYou ran out of attempts\e[0m\n");
    }
}
```

---

## Options

The options that the program supports include:
- show all fields without being requested
- evaluate guess case-sensitively
- different delimiter (e.g. tab for tsv or semicolon)
- choose selected data row
- limit number of guesses
- choose custom srand seed

The command line flags are interpreted by the **getopt()** function as follows:
```c
int main(int argc, char *argv[]) {
    // Create variables for command line flags
    char sepchar = ',';
    unsigned int show_all = 0, use_case = 0, setline = 0, seed = time(NULL) ^ getpid();
    int max_attempts = -1;
    int opt;
    char *end = NULL;

    // Parse flags
    while ((opt = getopt(argc, argv, "acd:hl:n:s:")) != -1) {
        switch (opt) {
            case 'a': show_all = 1; break;
            case 'c': use_case = 1; break;
            case 'd': sepchar = optarg[0]; break;
            case 'h': usage(); return 0;
            case 'l': setline = strtol(optarg, &end, 10); break;
            case 'n': max_attempts = strtol(optarg, &end, 10); break;
            case 's': seed = strtol(optarg, &end, 10); break;
            case '?': usage(); return 1;
        }
    }

    if (end && *end != '\0') {
        printf("Expected integer but recieved string.\n");
        return 1;
    }
    if (max_attempts < -1) {
        printf("Max attempts must be a positive number.\n");
        return 1;
    }
    if (setline < 0) {
        printf("Line number must be positive.\n");
        return 1;
    }

    srand(seed);

    char *filename;
    // if there are no non-flag arguments, return 1
    if (optind >= argc) {
        printf("No file input. Please enter the file path of the dataset.\n");
        char input[MAXOPL];
        if (!fgets(input, MAXOPL, stdin)) {
            printf("\n");
            return 1;
        }
        input[strcspn(input, "\n")] = '\0';
        filename = input;
    }
    else {
        filename = argv[optind];
    }

    // open the first non-flag argument
    FILE *file = fopen(filename, "r");
    if (!file) {
        perror("fopen");
        return 1;
    }
    // Parse first line for a linked list of header options
    struct Node *ops = node_alloc("");
    int nops = parse_line(ops, file, sepchar);

    // Choose line from file to be the secret
    int line = choose_line(file, setline);

    // parse the secret line
    struct Node *secret = node_alloc("");
    // if the length of the secret line is not the same as the header line,
    // throw error
    if (nops != parse_line(secret, file, sepchar)) {
        printf("Malformed line at line %d\n", line);
        cleanup(ops, secret);
        return 1;
    }

    // close file - no longer needed
    fclose(file);

    // let user try to guess secret item
    guess_loop(ops, secret, nops, show_all, use_case, max_attempts);
    // free both linked lists
    cleanup(ops, secret);
}

// Function to show help message
void usage() {
    printf(
        "Usage:\n"
        "  guess [-a] [-c] [-d CHAR] [-h] [-l LINE] [-n MAX] file\n"
        "\n"
        "  -a        show all fields at once\n"
        "  -c        case-sensitive answer matching (default: insensitive)\n"
        "  -d CHAR   field delimiter (default: ',')\n"
        "  -h        show this help message\n"
        "  -l LINE   use the nth data row after the header instead of a random one\n"
        "  -n MAX    limit number of guesses before revealing the answer\n"
        "  -s SEED   set a set seed for the random number generation\n"
    );
}
```

# Conclusion
In conclusion, this program is compatible with almost any csv file and opens a range of possibilities and different use cases.
