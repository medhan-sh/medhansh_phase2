# 1. buffer overflow 0

> Let's start off simple, can you overflow the correct buffer? The program is available here. You can view source here.
Connect using:
nc saturn.picoctf.net 51336


## Solution:

- The code had a vulnerability of not defining the value for input 
```
void vuln(char *input){
  char buf2[16];
  strcpy(buf2, input);
}
```
- The `flag` variable was already loaded into memory at the start of `main()`:
```
FILE *f = fopen("flag.txt","r");
fgets(flag, FLAGSIZE_MAX, f);  // Flag is now in memory
```
- The function prints `stout` if the program fails :
```
void sigsegv_handler(int sig) {
  printf("%s\n", flag);
  fflush(stdout);
  exit(1);
}
```
- So if i overflow the buffer value then the program would give me the flag

<img width="1280" height="832" alt="Screenshot 2025-10-30 at 10 54 50 PM" src="https://github.com/user-attachments/assets/5d09e522-2764-4d30-814c-44b2b438f689" />


## Flag:

```
picoCTF{ov3rfl0ws_ar3nt_that_bad_9f2364bc}
```

## Concepts learnt:

- How local varriables are stored.
- Signal handlers a function which operates when the program fails.
- The vulnerability with using unsafe functions like `strcpy` without bounds checking 
## Notes:

- NONE

## Resources:

- https://www.youtube.com/watch?v=1S0aBV-Waeo - a vid on buffer overflow


***

