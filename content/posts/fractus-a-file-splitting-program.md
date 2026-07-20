+++
date = '2026-07-20T13:28:42+01:00'
draft = false
title = 'Fractus - A File Splitting Program'
+++
# Fractus - A File Splitting Program
[Fractus](https://github.com/ZhaolunYin/fractus) is a program that splits a *secret file* into 15 files, requiring **all 15 files** to reconstruct the original. It does this by first hex encoding the **filename** and **each character** in the file, then generating 15 different files where no hex character in the *same place* as the original file or any other of the 15 files is the *same*. This way, the original file can be found by looking at the *missing hex char* from each file.

---

## Example

```txt
hello.txt
Hello, world
```
After it is hex encoded, it becomes:
```txt
68 65 6C 6C 6F 2E 74 78 74
48 65 6C 6C 6F 2C 20 77 6F 72 6C 64
```
Then, the 15 files created would be
```txt
unum.fractus
2C 23 88 13 45 47 29 F0 6B
AB 20 D3 24 78 8D 0A 80 84 CE 7B D6

duo.fractus
EA 39 FF 2E E2 B9 16 AE 91
0E CC 44 7F 87 C7 15 B5 57 EC C4 25

tres.fractus
91 87 D2 F7 06 EA 45 DF 2E
3D 79 F2 D6 CB 11 A7 2F E3 31 F3 C8

quattuor.fractus
14 B0 B4 EF 8E 1D C2 4A E2
25 AD 88 B5 B3 EF 9D F8 3A 57 1A AC

quique.fractus
7F 76 A1 9B 3B 53 FB 25 F3
86 0A A9 4E 95 A5 F4 DC 1E BD A2 1A

sex.fractus
4E 92 CB 01 FD 84 BA EC 38
C2 51 EB AB 26 D2 42 42 C1 8A B9 0D

septem.fractus
C9 EA ED B0 93 C2 DE 33 C7
F3 16 70 E1 EE 3A 3E C6 75 A0 0F 9F

octo.fractus
83 D1 3E 85 DC 6F 67 57 AA
BF 38 26 59 40 5E E8 93 0B 2B 45 30

novem.fractus
5D 1F 07 A9 51 90 E3 84 8F
90 E7 1E 82 0D 0B C1 61 D8 63 3E 49

decem.fractus
A7 4D 45 5D B9 05 0C BD 10
DC 8F C1 08 51 B3 BB 19 2D 19 9D 7B

undecim.fractus
02 C4 5A D6 2A D6 9D 09 B6
EA B3 97 CD D4 F4 63 3D 46 0F 88 5E

duodecim.fractus
D6 58 76 42 74 F8 38 62 05
57 4E 0A 3A F2 49 79 04 90 48 E6 E2

tredecim.fractus
B0 FC 10 38 10 AC 50 CB 5C
19 D4 B5 F3 1A 96 8F 5B AC 94 D0 B1

quattuordecim.fractus
35 0E 23 C4 A7 71 AF 16 D9
71 FB 5D 10 AC 78 56 AE B2 F6 27 83

quindecim.fractus
FB AB 99 7A C8 3B 81 91 4D
64 92 3F 97 39 60 DC EA F9 D5 51 F7
```
In this way, the original digit is *missing* from all the files, so it can be recreated once you posess all the fractus files. 

---

# Hex encode/decode
These 2 basic functions convert a single digit of hex to decimal and back.
```c
// Convert int (0-15) to hex digit
char int_to_hex(int i) {
    if (i >= 0 && i <= 9) // if 0-9 then return 0-9
        return '0' + i;
    else if (i >= 10 && i <= 15) // if 10-15 then return A-F
        return 'A' - 10 + i;
    else // else return null char
        return '\0';
}

// Convert hex digit (0-F) to int (0-15)
int hex_to_int(char x) {
    if (x >= '0' && x <= '9') // if 0-9, return digit
        return x - '0';
    else if (x >= 'A' && x <= 'F') // if 10-15, return A-F
        return x - 'A' + 10;
    else // else return -1
        return -1;
}
```
The **char_to_hex()** function fills a length 4 character array with 2 hex digits, a space and a null char from character c.
The **hex_to_char()** function does the reverse.
```c
// Convert char to hex string
// e.g. 'A' -> "41 "
void char_to_hex(unsigned char c, char *out) {
    out[0] = int_to_hex(c / 16); // 1st hex digit
    out[1] = int_to_hex(c % 16); // 2nd hex digit
    out[2] = ' '; // space
    out[3] = '\0'; // null char
}

// Convert hex to char
// e.g. "41 " -> 'A'
char hex_to_char(char *s) {
    // Return first char * 16 + 2nd char
    return (hex_to_int(s[0]) * 16) + hex_to_int(s[1]);
}
```

### Files
The **file_to_hex()** and **hex_to_file()** functions loop over a file stored as a character array and call *char_to_hex()* and *hex_to_char()* respectively.
```c
// Function to convert each char of a file to hex
void file_to_hex(char *filename, char **buffer) {
    // Open file in read binary mode
    FILE *rfile = fopen(filename, "rb");
    if (!rfile) {
        perror("fopen");
        free(*buffer);
        exit(1);
    }
    // Strip path
    if (char *no_path = strrchr(filename, '/')) {
        filename = no_path + 1;
    }
    // Encode filename as 1st line of file
    for (; *filename; filename++) {
        char s[4];
        char_to_hex(*filename, s);
        buf_add(buffer, s);
    }
    buf_add(buffer, "\n");
    int c;
    // While char read is not EOF (End Of File)
    while ((c = fgetc(rfile)) != EOF) {
        // if the char isn't a newline
        if (c != '\n') {
            // create a length 4 char array for char_to_hex
            char s[4];
            // convert char read into hex digits stored in s
            char_to_hex(c, s);
            // push s onto file buffer
            buf_add(buffer, s);
        }
        else {
            // print newlines as-is
            buf_add(buffer, "\n");
        }
    }
    // close file
    fclose(rfile);
}

// Convert a string produced by file_to_hex back into the original file
void hex_to_file(char *buffer) {
    // Create a read buffer for 3 chars (2 hex + space)
    char buf[3];
    // Create a filename buffer
    char *filename = malloc(sizeof(char));
    filename[0] = '\0';
    // Read the 1st line of the buffer to get the filename
    for (; *buffer && *buffer != '\n'; buffer += 3) {
        char add[2] = { hex_to_char(buffer), '\0'};
        buf_add(&filename, add);
    }
    // Increment buffer by 1 to skip newline
    buffer++;
    // strip path
    if (char *no_path = strrchr(filename, '/')) {
        filename = no_path + 1;
    }
    // open file in write binary mode
    FILE *fptr = fopen(filename, "wb");
    if (!fptr) {
        perror("fopen");
        return;
    }
    // loop through each char in the buffer
    for (; *buffer; buffer += 3) {
        // if the buffer has a newline
        if (*buffer == '\n') {
            // print as is
            fprintf(fptr, "\n");
            // subtract 2 from the buffer so when
            // it adds 3 at the end of the loop
            // it has the net effect of doing buffer++
            // instead
            buffer -= 2;
        }
        else {
            // decode the hex value and print the char
            fprintf(fptr, "%c", hex_to_char(buffer));
        }
    }
    // close the file pointer and free the filename buffer
    fclose(fptr);
    free(filename);
}
```
### Buffer management
The character arrays are managed using the *buf_add()* function, which concatenates 2 strings.
```c
// Function to concatentate 2 strings
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
---
# File splitting
### Filenames
The files are named after the numbers 1-15 in latin.
```c
// Array of filenames
const char *filenames[N_FILES] = {
    "unum.fractus", "duo.fractus", "tres.fractus",
    "quattuor.fractus", "quinque.fractus", "sex.fractus",
    "septem.fractus", "octo.fractus", "novem.fractus",
    "decem.fractus", "undecim.fractus", "duodecim.fractus",
    "tredecim.fractus", "quattuordecim.fractus", "quindecim.fractus"
};
```
### Split/Merge functions
The file is split using the **split()** function and reconstructed using the **merge()** function.
```c
// Function to split a file stored in a char *
// into 15 files which are completely different from each other
// (no non-space char is the same in each position)
void split(char *file) {
    // handle null buffer
    if (!file) {
        fprintf(stderr, "NULL buffer");
        return;
    }
    art(file);
    // create 15 file pointers
    FILE *fptr[N_FILES];
    int opened = 0;
    // Open each file
    for (int i = 0; i < N_FILES; i++) {
        fptr[i] = fopen(filenames[i], "wb");
        if (!fptr[i]) {
            perror("fopen");
            for (int j = 0; j < opened; j++) {
                fclose(fptr[j]);
            }
            return;
        }
        opened++;
    }
    // loop through each char in file
    for (; *file; file++) {
        // if space or newline, print as is
        if (*file == ' ' || *file == '\n') {
            for (int i = 0; i < N_FILES; i++) {
                fprintf(fptr[i], "%c", *file);
            }
            continue;
        }
        // create list of all hex values NOT in the file
        int list[N_FILES];
        for (int i = 0, j = 0; i < 16; i++) {
            if (i != hex_to_int(*file) && hex_to_int(*file) != -1) {
                list[j++] = i;
            }
        }
        // shuffle list using Fisher-Yates algorithm
        for (int i = N_FILES - 1; i > 0; i--) {
            int j = rand() % (i + 1);
            int t = list[i];
            list[i] = list[j];
            list[j] = t;
        }
        // Print shuffled hex digit in each file
        for (int i = 0; i < N_FILES; i++) {
            char c = int_to_hex(list[i]);
            fprintf(fptr[i], "%c", c);
        }
    }
    // Close files
    for (int i = 0; i < N_FILES; i++) {
        fclose(fptr[i]);
    }
}

// Function to re-merge files created from split() to obtain original file
void merge(char **buffer, int keep) {
    // Open the 15 files
    FILE *fptr[N_FILES];
    int opened = 0;
    for (int i = 0; i < N_FILES; i++) {
        fptr[i] = fopen(filenames[i], "rb");
        if (!fptr[i]) {
            perror("fopen");
            for (int j = 0; j < opened; j++) {
                fclose(fptr[j]);
            }
            return;
        }
        opened++;
    }
    int eof_count = 0;
    // While true
    while (!eof_count) {
        int total = 0;
        int c;
        int read = 1;
        // Read a digit from each file
        for (int i = 0; i < N_FILES; i++) {
            // if digit is EOF, exit
            if ((c = fgetc(fptr[i])) == EOF) {
                eof_count++;
                continue;
            }
            // if it's a space or newline, finish the loop without reading
            if (read) {
                if (c == '\n' || c == ' ') {
                    char buf[2] = { c, '\0' };
                    buf_add(buffer, buf);
                    read = 0;
                }
                else {
                    // add hex value to total
                    total += hex_to_int(c);
                }
            }
        }
        // if total is not 0
        if (total) {
            // find missing value by subtracting from total
            int diff = HEX_SUM - total;
            char hex = int_to_hex(diff);
            if (!hex) {
                fprintf(stderr, "Files are corrupted\n");
                for (int i = 0; i < N_FILES; i++) {
                    fclose(fptr[i]);
                }
                return;
            }
            // push hex onto buffer
            char buf[2] = { hex, '\0' };
            buf_add(buffer, buf);
        }
    }
    // If not all files reached EOF, print error
    if (eof_count != 15) {
        fprintf(stderr, "Files have different lengths");
    }
    // Closed all files
    for (int i = 0; i < N_FILES; i++) {
        fclose(fptr[i]);
        if (!keep) {
            remove(filenames[i]);
        }
    }
    art(*buffer);
}
```
---
# Randomart
Randomart similar to the ones produced by ssh-keygen are produced by the **art()** function, which hashes the string using the **djb2** algorithm and generates randomart using the **drunken bishop** algorithm.
```c
// Randomart generation using drunken bishop algorithm
void art(char *s) {
    const char *symbols[] = {
        " ",
        "+",
        "┼",
        "╪",
        "╫",
        "╬",
        "╳",
    };
    const char *start_end[] = { "▚", "▞" };
    int n_symbols = sizeof(symbols) / sizeof(symbols[0]);
    int x = ART_WIDTH / 2;
    int y = ART_HEIGHT / 2;
    int count[ART_HEIGHT][ART_WIDTH] = { 0 };
    // Generate hash using djb2 algorithm
    unsigned long hash = 5381;
    for (char *c = s; *c; c++) {
        hash = ((hash << 5) + hash) + *c;
    }
    count[y][x] = -1;
    // For every 2 bits in the hash
    for (int i = 0; i < (int)(sizeof(hash) * 8); i += 2) {
        // Switch from first 2 bits
        switch ((hash >> i) & 0x3) {
            case 0:
                x = (x > 0) ? x - 1 : x;
                y = (y > 0) ? y - 1 : y;
                break;
            case 1:
                x = (x < ART_WIDTH - 1) ? x + 1 : x;
                y = (y > 0) ? y - 1 : y;
                break;
            case 2:
                x = (x > 0) ? x - 1 : x;
                y = (y < ART_HEIGHT - 1) ? y + 1 : y;
                break;
            case 3:
                x = (x < ART_WIDTH - 1) ? x + 1 : x;
                y = (y < ART_HEIGHT - 1) ? y + 1 : y;
                break;
            default:
                break;
        }
        count[y][x] = (count[y][x] >= 0 && count[y][x] < n_symbols - 1) ? count[y][x] + 1 : count[y][x];
    }
    count[y][x] = -2;
    printf("╔");
    for (int i = 0; i < ART_WIDTH; i++) {
        printf("═");
    }
    printf("╗\n");
    for (int i = 0; i < ART_HEIGHT; i++) {
        printf("║");
        for (int j = 0; j < ART_WIDTH; j++) {
            printf("%s", (count[i][j] >= 0) ? symbols[count[i][j]] : start_end[(count[i][j] * -1) - 1]);
        }
        printf("║\n");
    }
    printf("╚");
    for (int i = 0; i < ART_WIDTH; i++) {
        printf("═");
    }
    printf("╝\n");
}
```
---
